---
title: 'Functional profiling with nf-core/funcprofiler'
---

:::::::::::::::::::::::::::::::::::::: questions

- How do I configure and run nf-core/funcprofiler?
- What do HUMAnN and eggNOG-mapper measure?
- What is the difference between gene-family abundance, pathway abundance and pathway coverage?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Configure functional profiling with nf-core/funcprofiler
- Explain the roles of HUMAnN and eggNOG-mapper
- Distinguish gene-family abundance, pathway abundance and pathway coverage
- Identify the standardised function tables needed for downstream analysis

::::::::::::::::::::::::::::::::::::::::::::::::

## HUMAnN and eggNOG-mapper

- HUMAnN
- eggNOG-mapper


## Input: pre-processed reads

nf-core/funcprofiler does not perform read QC or host removal. We use the reads saved by taxprofiler with `--save_analysis_ready_fastqs`. We work in the same folder as in the previous episodes, where the database sheets and the Nectar config file are:

```bash
cd /mnt/data/taxprofiler
ls /mnt/data/taxprofiler/results/analysis_ready_fastqs/
```

fastp names the cleaned reads `<sample>_<run_accession>_1.fastp.fastq.gz` and `<sample>_<run_accession>_2.fastp.fastq.gz`:

```bash
READS=/mnt/data/taxprofiler/results/analysis_ready_fastqs
echo "sample,run_accession,instrument_platform,fastq_1,fastq_2,fasta" > samplesheet_func.csv
for i in 1 2 3 4 5; do
  r1=$READS/gut${i}_run1_1.fastp.fastq.gz
  r2=$READS/gut${i}_run1_2.fastp.fastq.gz
  echo "gut${i},run1,ILLUMINA,${r1},${r2}," >> samplesheet_func.csv
done
cat samplesheet_func.csv
```

Check that every file listed exists:

```bash
tail -n +2 samplesheet_func.csv | cut -d, -f4,5 | tr ',' '\n' | xargs ls -l
```

## Running the pipeline

During the workshop we run DIAMOND, the only database in `databases_funcprofiler.csv`. HUMAnN 4 and eggNOG-mapper need much larger databases and several hours, so their outputs have been precomputed for you.

```bash
nextflow run nf-core/funcprofiler \
  -r 1.0.0 \
  -profile docker \
  -c configs/nectar.config \
  --input samplesheet_func.csv \
  --databases databases_funcprofiler.csv \
  --outdir /mnt/data/funcprofiler/results \
  --run_diamond \
  -resume
```

## Outputs

```bash
ls /mnt/data/funcprofiler/results
ls /mnt/data/funcprofiler/results/diamond/gut_workshop_v1
ls /mnt/data/funcprofiler/results/humann4
```

<!-- TODO: paste real output -->

All outputs are in `/mnt/data/funcprofiler/results/`:

- MultiQC report: `multiqc/multiqc_report.html`
- DIAMOND alignments (run during the workshop): `diamond/gut_workshop_v1/`
- HUMAnN 4 outputs (precomputed), one set of files per sample in `humann4/`:
  - Gene family abundance: `<sample>_2_genefamilies.tsv.gz`
  - Reaction abundance: `<sample>_3_reactions.tsv.gz`
  - Pathway abundance: `<sample>_4_pathabundance.tsv.gz`
  - Gene families regrouped to other functional categories: `<sample>_regroup.tsv.gz`
  - Species functional profiles: stratified rows (`PATHWAY|g__Genus.s__species`) in the HUMAnN tables
- eggNOG-mapper annotations (precomputed): `eggnogmapper/*.emapper.annotations`
- Execution and resource reports: `pipeline_info/`

HUMAnN 4 does not write a separate pathway coverage table as HUMAnN 3 did.

```bash
zcat /mnt/data/funcprofiler/results/humann4/Gut_sample1_4_pathabundance.tsv.gz | head -n 5
```

The precomputed outputs use the sample names `Gut_sample1` to `Gut_sample5` instead of `gut1` to `gut5`. A copy of the HUMAnN 4 outputs is also in `/mnt/data/workshop_data/funcprofiler/humann4/`, which is the folder we use in R.

::::::::::::::::::::::::::::::::::::: keypoints

- nf-core/funcprofiler expects reads that have already been through QC.
- HUMAnN reports gene families, pathway abundance and pathway coverage, overall and per species.

::::::::::::::::::::::::::::::::::::::::::::::::
