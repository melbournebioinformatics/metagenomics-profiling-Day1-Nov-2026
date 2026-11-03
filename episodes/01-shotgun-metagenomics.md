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
