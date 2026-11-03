---
title: 'Shotgun metagenomics data to study the microbiome'
---

:::::::::::::::::::::::::::::::::::::: questions

- What is the human gut microbiome and how can we study it with sequencing?
- How does shotgun metagenomics differ from marker-gene (16S rRNA) sequencing?
- What is the difference between taxonomic and functional profiling?
- Which dataset will we use in this workshop, and how do we obtain it?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Understand the general concepts of the human gut microbiome
- Distinguish shotgun metagenomics from marker-gene sequencing
- Explain the difference between taxonomic and functional profiling
- Map the main stages from FASTQ files to biological interpretation
- Describe the workshop dataset and metadata
- Download the workshop dataset and verify its integrity

::::::::::::::::::::::::::::::::::::::::::::::::

## The human gut microbiome

## Shotgun metagenomics and marker-gene sequencing

- Marker-gene sequencing amplifies and sequences one gene region (e.g. 16S rRNA, ITS).
- Shotgun metagenomics sequences all DNA in the sample.

## Taxonomic and functional profiling

- Taxonomic profiling: which organisms are present and their abundance.
- Functional profiling: which genes and metabolic pathways are present and their abundance.

## The workshop dataset

5 simulated human gut metagenomes from Sun, Huang et al. (2021), with paired-end reads, ground-truth abundance tables and sample metadata. The true composition of each sample is known.

## Downloading the data and checking its integrity

The reads are already on the Nectar instance, so we do not download them today. With your own projects you will usually start by downloading the data from a public repository, and the first thing to do after a download is to check that the files arrived complete and unchanged.

::::::::::::::::::::::::::::::::::::: callout

### We do not run this today

The commands in this section show how you would download and check the data in any other situation. The full dataset is about 6.5 GB, which would take too long during the workshop.

::::::::::::::::::::::::::::::::::::::::::::::::

### Downloading with `wget`

The full gut samples are in the figshare dataset [Gut_sample](https://doi.org/10.6084/m9.figshare.12234947.v1) (figshare project 79916). Each file on figshare has its own download link, which you can copy from the file's download button. `wget` downloads a file from a link, and `-O` sets the name of the saved file:

```bash
mkdir -p ~/raw_data && cd ~/raw_data
wget -O Gut_sample1.fq.gz "https://figshare.com/ndownloader/files/<file_id>"
```

For sequencing data from the European Nucleotide Archive (ENA), the [ENA browser](https://www.ebi.ac.uk/ena/browser/) lists the FTP link of every FASTQ file of a project, and `wget` works the same way.

### What an MD5 checksum is

An MD5 checksum is a 32-character code calculated from the content of a file. If a single byte of the file changes, for example because a download was cut short, the checksum changes completely. Repositories publish the checksum of each file they host, so you can calculate the checksum of your copy and compare the two. figshare shows the MD5 of each file on the dataset page and in its API, and ENA lists it in the `fastq_md5` column of a project's file report.

`md5sum` calculates the checksum of one or more files:

```bash
md5sum Gut_sample1.fq.gz
```

```output
<32-character checksum>  Gut_sample1.fq.gz
```

### Checking many files at once

Put the published checksums in a text file, one line per file, with the checksum, two spaces and the file name:

```output
<checksum of file 1>  Gut_sample1.fq.gz
<checksum of file 2>  Gut_sample2.fq.gz
```

Then `md5sum -c` recalculates the checksum of every file in the list and compares it with the published one:

```bash
md5sum -c md5sums.txt
```

```output
Gut_sample1.fq.gz: OK
Gut_sample2.fq.gz: OK
```

Any file reported as `FAILED` is damaged or incomplete and should be downloaded again. Keep `md5sums.txt` with your raw data: it lets you, or anyone else, check later that the files have not changed.

## First look at the reads

The reads for this workshop are already on the Nectar instance, in `/mnt/data/rawreads/`. There are two compressed FASTQ files per sample: `_R1` and `_R2` hold the two reads of each pair.

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 1: Look inside a FASTQ file

1. List the files in `/mnt/data/rawreads/` and check their sizes.
2. Print the first two reads of `Gut_sample1_sub_R1.fastq.gz`. The files are compressed, so use `zcat` instead of `cat`.

:::::::::::::::::::::::: solution

```bash
ls -lh /mnt/data/rawreads/
zcat /mnt/data/rawreads/Gut_sample1_sub_R1.fastq.gz | head -n 8
```

Each read takes four lines: a header starting with `@`, the bases, a `+` separator, and the quality of each base as one character per base.

:::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

## How many reads are there?

Each read takes four lines, so the number of reads in a file is the number of lines divided by four:

```bash
echo $(( $(zcat /mnt/data/rawreads/Gut_sample1_sub_R1.fastq.gz | wc -l) / 4 ))
```

```output
100000
```

A loop does the same for every R1 file:

```bash
for f in /mnt/data/rawreads/*_sub_R1.fastq.gz; do
  echo "$(basename "$f") $(( $(zcat "$f" | wc -l) / 4 ))"
done
```

```output
Gut_sample1_sub_R1.fastq.gz 100000
Gut_sample2_sub_R1.fastq.gz 100000
Gut_sample3_sub_R1.fastq.gz 100000
Gut_sample4_sub_R1.fastq.gz 100000
Gut_sample5_sub_R1.fastq.gz 100000
```

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 2: Do the pairs match?

The R1 and R2 files of a sample must contain the same number of reads, because each read in R1 has its partner in R2. Check this for every sample.

:::::::::::::::::::::::: solution

```bash
for r1 in /mnt/data/rawreads/*_sub_R1.fastq.gz; do
  r2=${r1/_R1/_R2}
  n1=$(( $(zcat "$r1" | wc -l) / 4 ))
  n2=$(( $(zcat "$r2" | wc -l) / 4 ))
  echo "$(basename "$r1" _sub_R1.fastq.gz) R1=$n1 R2=$n2"
done
```

Every sample should show the same number for R1 and R2. A difference means one of the files is truncated or corrupted.

:::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Shotgun metagenomics sequences all DNA in a sample.
- Taxonomic profiling describes which organisms are present; functional profiling describes their genes and pathways.
- The workshop uses 5 simulated gut metagenomes with a known composition.

::::::::::::::::::::::::::::::::::::::::::::::::
