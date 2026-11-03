---
title: 'Taxonomic profiling with nf-core/taxprofiler'
teaching: 45
exercises: 25
---

:::::::::::::::::::::::::::::::::::::: questions

- How do I configure and run nf-core/taxprofiler?
- How do sylph, mOTUs, Kraken2/Bracken and MetaPhlAn differ?
- How do I interpret the pipeline outputs?
- Which table do I need for downstream analysis in R?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Configure taxonomic profiling with nf-core/taxprofiler
- Explain the roles of sylph, mOTUs, Kraken2/Bracken and MetaPhlAn
- Interpret the MultiQC report, classification summaries and pipeline reports
- Identify the standardised abundance table needed for downstream analysis

::::::::::::::::::::::::::::::::::::::::::::::::

## The profilers

- Kraken2 and Bracken
- MetaPhlAn (we will not run this profiler during the workshop due, precomputed outputs will be provided)
- mOTUs
- sylph (we will not run this profiler during the workshop due, precomputed outputs will be provided)

## Running the pipeline

```bash
cd /mnt/data/taxprofiler
./nextflow_launch.sh
```

or directly:

```bash
nextflow run nf-core/taxprofiler \
  -r 2.0.1 \
  -profile docker \
  --input samplesheet.csv \
  --databases databases_taxprofiler.csv \
  --outdir /mnt/data/taxprofiler/results \
  --perform_shortread_qc \
  --save_analysis_ready_fastqs \
  --run_kraken2 --run_bracken \
  --run_motus \
  --run_profile_standardisation \
  --run_krona \
  -resume
```

## Output overview

```bash
ls /mnt/data/taxprofiler/results
```

```output
analysis_ready_fastqs  bracken  fastp  kraken2  krona  motus
multiqc  pipeline_info  
```

### MultiQC report

`multiqc/multiqc_report.html` contains the read QC (fastp) and classification summaries. Copy it to your laptop to open it:

```bash
# PENDING TO WRITE CODE
```

### Taxonomic classification summaries
The pipeline writes one folder per profiler into `results/`:

```text
results/
├── kraken2
│   ├── k2_standard_08_GB_20260626
│   │   ├── Gut_sample1_Gut_sample1_run1_k2_standard_08_GB_20260626.kraken2.kraken2.report.txt
│   │   ├── ...                                   (one report per sample)
│   │   └── Gut_sample5_Gut_sample5_run1_k2_standard_08_GB_20260626.kraken2.kraken2.report.txt
│   ├── k2_standard_08_GB_20260626_bracken
│   ├── kraken2_k2_standard_08_GB_20260626_bracken-bracken_combined_reports.txt
│   └── kraken2_k2_standard_08_GB_20260626_combined_reports.txt
├── bracken
│   ├── bracken_k2_standard_08_GB_20260626_bracken_combined_reports.txt
│   └── k2_standard_08_GB_20260626_bracken
│       ├── Gut_sample1_..._bracken.bracken.kraken2.report_bracken.txt
│       ├── Gut_sample1_..._bracken.bracken.tsv
│       └── ...                                   (two files per sample)
├── motus
│   ├── db_mOTU_v3.1.0
│   │   ├── Gut_sample1_Gut_sample1_run1_db_mOTU_v3.1.0.log
│   │   ├── Gut_sample1_Gut_sample1_run1_db_mOTU_v3.1.0.out
│   │   └── ...                                   (.log + .out per sample)
│   └── motus_db_mOTU_v3.1.0_combined_reports.txt
└── taxpasta
    ├── bracken_k2_standard_08_GB_20260626_bracken.tsv
    ├── kraken2_k2_standard_08_GB_20260626_bracken-bracken.tsv
    ├── kraken2_k2_standard_08_GB_20260626.tsv
    ├── metaphlan_mpa_vJun23_CHOCOPhlAnSGB_202403.tsv
    └── motus_db_mOTU_v3.1.0.tsv
```

### Standardised and merged tables (Taxpasta)

Taxpasta converts the output of each profiler into one merged table per profiler and database, with taxa in rows and samples in columns. The tables are in `taxpasta/`.

```bash
ls /mnt/data/taxprofilertaxprofiler/results/taxpasta/
```

We import these tables into R in the episode Importing and exploring microbiome data with R.

### Pipeline execution and resource reports

`pipeline_info/` contains the execution report, timeline, trace and software versions.

::::::::::::::::::::::::::::::::::::: keypoints

- nf-core/taxprofiler runs several profilers on the same pre-processed reads.
- The Taxpasta tables are the input for the analysis in R.
- `pipeline_info/` records resource usage and software versions.

::::::::::::::::::::::::::::::::::::::::::::::::
