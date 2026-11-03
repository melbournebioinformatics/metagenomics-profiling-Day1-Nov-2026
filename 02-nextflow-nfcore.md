---
title: 'Reproducible workflows with Nextflow and nf-core'
---

:::::::::::::::::::::::::::::::::::::: questions

- Why is reproducibility a challenge in bioinformatics?
- What are Nextflow and nf-core, and how do they help?
- How do I connect to a Nectar instance and check that the software is available?
- How do I prepare the input files needed by nf-core/taxprofiler and nf-core/funcprofiler?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Understand reproducibility in bioinformatics
- Explain what Nextflow is and why it is useful
- Navigate the nf-core website and find suitable pipelines
- Connect to a Nectar instance
- Load and check the software used in the workshop
- Construct reproducible commands for both pipelines
- Prepare and validate samplesheets and database sheets

::::::::::::::::::::::::::::::::::::::::::::::::

## Reproducibility in bioinformatics

## Nextflow and nf-core

- Nextflow: workflow manager.
- nf-core: community-curated Nextflow pipelines.

Pipelines are listed on the nf-core website. In this workshop we use:

- nf-core/taxprofiler
- nf-core/funcprofiler

## Connecting to Nectar

```bash
ssh ubuntu@<IP_adress>
```
::::::::::::::::::::::::::::::::::::: callout
We already prepared the databases for this workshop. If you are following the workshop remotely, please download the databases 
for each chosen tool as well as modify the path accordingly.
:::::::::::::::::::::::::::::::::::::

## Loading the software

Check that each tool is available:

```bash
java -version
nextflow -version
apptainer --version
nf-core --version
```

## Preparing the samplesheet

Both pipelines use the same samplesheet columns: `sample,run_accession,instrument_platform,fastq_1,fastq_2,fasta`. Paths must be absolute.

```bash
cd /mnt/data/taxprofiler

echo "sample,run_accession,instrument_platform,fastq_1,fastq_2,fasta" > samplesheet.csv
for i in 1 2 3 4 5; do
  echo "gut${i},run1,ILLUMINA,/mnt/data/rawreads/Gut_sample${i}_sub_R1.fastq.gz,/mnt/data/rawreads/Gut_sample${i}_sub_R2.fastq.gz," >> samplesheet.csv
done
cat samplesheet.csv
```



```output
sample,run_accession,instrument_platform,fastq_1,fastq_2,fasta
gut1,run1,ILLUMINA,/mnt/data/rawreads/Gut_sample1_sub_R1.fastq.gz,/mnt/data/rawreads/Gut_sample1_sub_R2.fastq.gz,
gut2,run1,ILLUMINA,/mnt/data/rawreads/Gut_sample2_sub_R1.fastq.gz,/mnt/data/rawreads/Gut_sample2_sub_R2.fastq.gz,
gut3,run1,ILLUMINA,/mnt/data/rawreads/Gut_sample3_sub_R1.fastq.gz,/mnt/data/rawreads/Gut_sample3_sub_R2.fastq.gz,
gut4,run1,ILLUMINA,/mnt/data/rawreads/Gut_sample4_sub_R1.fastq.gz,/mnt/data/rawreads/Gut_sample4_sub_R2.fastq.gz,
gut5,run1,ILLUMINA,/mnt/data/rawreads/Gut_sample5_sub_R1.fastq.gz,/mnt/data/rawreads/Gut_sample5_sub_R2.fastq.gz,
```

Check that every file listed exists:

```bash
tail -n +2 samplesheet.csv | cut -d, -f4,5 | tr ',' '\n' | xargs ls -l
```

## Preparing the database sheets

### taxprofiler: `databases_taxprofiler.csv`

Columns are `tool,db_name,db_params,db_type,db_path`.

```bash
cat > databases_taxprofiler.csv << 'EOF'
tool,db_name,db_params,db_type,db_path
kraken2,k2_standard_08_GB_20260626,,short;long,/mnt/data/taxprofiler/databases/kraken2/k2_standard_08_GB_20260626
bracken,k2_standard_08_GB_20260626_bracken,;-r 150,short,/mnt/data/taxprofiler/databases/kraken2/k2_standard_08_GB_20260626
motus,db_mOTU_v3.1.0,,short;long,/mnt/data/taxprofiler/databases/motus/db_mOTU
EOF
```


### funcprofiler: `databases_funcprofiler.csv`

Columns are `tool,db_name,db_entity,db_params,db_path`. HUMAnN and eggNOG-mapper need one row per database component.

```bash
cat > databases_funcprofiler.csv << 'EOF'
tool,db_name,db_entity,db_params,db_path
diamond,gut_workshop_v1,,,/mnt/data/funcprofiler/databases/diamond
EOF
```

## Set the pipeline’s resource limits
Nextflow does not know how many CPUs and how much memory your machine has. Therefore, a config file is needed to cap
every task at what your machine actually has, so no step asks for more and fails. We will create a config file for Nectar.
Again, if you are following this workshop remotely, modify the parameters in the following code that match with your machine.

```bash
mkdir -p configs

CPUS=$(nproc) #number of CPUs
MEM=$(( $(free -g | awk '/^Mem:/{print $2}') - 2 )) #number of RAM minus 2 (always spare some for trivial tasks)

cat > configs/nectar.config <<EOF
workDir = '/mnt/data/taxprofiler/work'
process {
    resourceLimits = [ cpus: ${CPUS}, memory: '${MEM}.GB', time: '48.h' ]
}
executor {
    name   = 'local'
    cpus   = ${CPUS}
    memory = '${MEM}.GB'
}
EOF
```


## Reproducible launch commands

Parameters go in a `params.json` file and the command in a launch script.

```bash
cat > params_taxprofiler.json << EOF
{
  "input": "/mnt/data/taxprofiler/samplesheet.csv",
  "databases": "/mnt/data/taxprofiler/databases_taxprofiler.csv",
  "outdir": "/mnt/data/taxprofiler/results",
  "perform_shortread_qc": true,
  "save_analysis_ready_fastqs": true,
  "run_kraken2": true,
  "run_bracken": true,
  "run_metaphlan": false,
  "run_motus": true,
  "run_sylph": false,
  "run_profile_standardisation": true,
  "run_krona": true,
  "taxpasta_add_name": false,
  "taxpasta_add_rank": false,
  "taxpasta_add_lineage": false
}
EOF
```

```bash
cat > nf_taxprofiler_launch.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail
nextflow run nf-core/taxprofiler \
  -r 2.0.1 \
  -profile docker \
  -c configs/nectar.config \
  -params-file params_taxprofiler.json \
  -resume
EOF
chmod +x nextflow_launch.sh
```

Alternatively, the nf-core tools launcher writes a `nf-params.json` interactively:

```bash
nf-core pipelines launch nf-core/taxprofiler -r <version>
```

::::::::::::::::::::::::::::::::::::: keypoints

- Pin pipeline versions with `-r`.
- Check software with `nextflow -version` and `apptainer --version` before launching.
- Both pipelines need a samplesheet with absolute paths and a database sheet.
- Keep parameters in `params.json` and the command in a launch script.

::::::::::::::::::::::::::::::::::::::::::::::::
