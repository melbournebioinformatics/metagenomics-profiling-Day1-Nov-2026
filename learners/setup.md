---
title: Setup
---

You do not need to install or download anything for this workshop. Everything runs on cloud instances on [Nectar](https://ardc.edu.au/services/ardc-nectar-research-cloud/)
that the instructors set up for you. You only need a laptop with:

- an **SSH client**, to connect to the Nectar instance on the command line. macOS and
  Linux have one in the Terminal, while on Windows use PowerShell, Windows Terminal or
  [MobaXterm](https://mobaxterm.mobatek.net/).
- an up-to-date **web browser**, to use RStudio Server and open the pipeline reports.

The instructors will give you the IP addresses, usernames and passwords on the day.

## Computing environments

| Part of the workshop | Where it runs | How you connect | What is pre-installed |
|---|---|---|---|
| Nextflow and nf-core pipelines (Day 1) | Nectar instance | SSH, command line | Java, Nextflow, Apptainer/Singularity, nf-core/taxprofiler, nf-core/funcprofiler, input data and reference databases |
| Analysis in R (Day 1 and Day 2) | RStudio Server on Nectar | Web browser | R, RStudio, all R packages and the pipeline outputs needed for both days |

### Connecting to the Nectar instance (command line)

```bash
ssh <username>@<instance-IP>
```

Enter the password when prompted. You should now see a prompt on the Nectar instance.

### Connecting to RStudio Server (R)

1. Open `http://<rstudio-IP>` in your web browser. The instructors will give you the address.
2. Log in with your username and password.
3. To check that everything is available, run in the RStudio console:

```r
library(mia)
library(miaViz)
packageVersion("mia")
list.files("/mnt/data/workshop_data")
```

You should see the mia version and the folders `funcprofiler`, `metadata` and
`taxprofiler`.

## Data

- 5 simulated human gut metagenomes, Illumina paired-end, from Sun, Huang et al. (2021)
  *Challenges in benchmarking metagenomic profilers*. Nature Methods 18:618–626.
  <https://doi.org/10.1038/s41592-021-01141-3>
- Source: figshare project 79916, dataset Gut_sample:
  <https://doi.org/10.6084/m9.figshare.12234947.v1>
- 10 FASTQ files (`Gut_sampleN.left.fq.gz` and `Gut_sampleN.right.fq.gz`) and 2
  ground-truth tables (`5_gut_taxonomic_abd.txt`, `5_gut_sequence_abd.txt`)

On the workshop instances:

- the raw reads are in `/mnt/data/taxprofiler/input/reads/`
- the pipeline outputs used in R are in `/mnt/data/workshop_data/`

## Reference databases

All databases are pre-installed under `/mnt/data/taxprofiler/databases/` and `/mnt/data/funcprofiler/databases/`:

- Kraken2 Standard-8 (`k2_standard_08_GB_20260626`)
- MetaPhlAn `mpa_vJun23_CHOCOPhlAnSGB_202403`
- mOTUs v3.1.0
- sylph GTDB r220 (`gtdb_r220_c200`)
- HUMAnN 4 databases
- eggNOG 5.0.2

:::::::::::::::::::::::::::::::::::::::::: spoiler

### Running the R analysis on your own computer after the workshop

The lesson uses R 4.6 and Bioconductor 3.23. To reproduce the R episodes on your own computer, install [R](https://cran.r-project.org/) and [RStudio Desktop](https://posit.co/download/rstudio-desktop/), then run:

```r
install.packages(c("readr", "dplyr", "tidyr", "ggplot2", "BiocManager"))
BiocManager::install(version = "3.23")
BiocManager::install(c("SummarizedExperiment", "TreeSummarizedExperiment",
                       "mia", "miaViz"))
```

You will also need a copy of the pipeline outputs. Change `data_dir` in the R episodes
to the folder where you saved them.

::::::::::::::::::::::::::::::::::::::::::::::::::
