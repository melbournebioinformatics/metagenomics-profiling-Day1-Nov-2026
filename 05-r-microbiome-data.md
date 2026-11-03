---
title: 'Importing and exploring microbiome data with R'
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- Which R data structures are used for microbiome data?
- How do I import a taxonomic profile, a functional profile and sample metadata into R?
- How do I aggregate, filter and transform microbiome data?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Identify the phyloseq and TreeSummarizedExperiment object types and explain their principal differences
- Import a MetaPhlAn taxonomic profile, HUMAnN pathway abundances and the sample metadata
- Build a TreeSummarizedExperiment object and explore its components
- Aggregate taxa at a selected rank, filter low-prevalence features and calculate relative abundance
- Visualise taxonomic composition, prevalence and the most abundant taxa and pathways

::::::::::::::::::::::::::::::::::::::::::::::::

In the previous episodes nf-core/taxprofiler and nf-core/funcprofiler produced several
tables. Here we bring one taxonomic profile (MetaPhlAn) and one functional profile
(HUMAnN pathway abundance) into R. We use them to build data objects that Bioconductor
packages can work with, and to take a first look at the samples.

::::::::::::::::::::::::::::::::::::::::: callout

### Where do we run R?

We use the workshop RStudio Server, a separate Nectar instance from the one where we ran the pipelines. Copies of the pipeline outputs are already in `/data/workshop_data/` on the RStudio Server, so we read them directly and do not need to copy anything.

```output
/data/workshop_data
├── funcprofiler/humann4/        # HUMAnN 4 outputs, one set of files per sample
├── metadata/                    # samplesheets used to run the pipelines
└── taxprofiler/                 # Taxpasta tables and merged profiler reports
```

::::::::::::::::::::::::::::::::::::::::::::::::

## Loading the libraries


``` r
library(readr)
library(dplyr)
library(tidyr)
library(ggplot2)
library(SummarizedExperiment)
library(TreeSummarizedExperiment)
library(mia)
library(miaViz)
```

Record the package versions. Function names in mia and miaViz have changed between
releases, so this helps when you compare your code with other tutorials:


``` r
packageVersion("mia")
```

``` output
[1] '1.20.0'
```

``` r
packageVersion("miaViz")
```

``` output
[1] '1.20.0'
```

Then define where the data are:


``` r
data_dir  <- "/data/workshop_data"
mpa_file  <- file.path(data_dir, "taxprofiler",
                       "metaphlan_mpa_vJun23_CHOCOPhlAnSGB_202403_combined_reports.txt")
meta_file <- file.path(data_dir, "metadata", "taxprofiler_samplesheet.csv")
humann_dir <- file.path(data_dir, "funcprofiler", "humann4")

# TRUE on the workshop RStudio Server, where all the input files are present.
# The lesson website is built without these files, so there the chunks that
# read the data are shown but not run.
has_data <- file.exists(mpa_file) &&
            file.exists(meta_file) &&
            length(list.files(humann_dir,
                              pattern = "_4_pathabundance\\.tsv\\.gz$")) > 0
```

## phyloseq and TreeSummarizedExperiment

A microbiome dataset has three parts that must stay in sync: an abundance table (features × samples), information about the features (taxonomy, pathway names) and information about the samples (metadata). Two R object types keep these together:

| Component              | phyloseq      | TreeSummarizedExperiment (TSE) |
|------------------------|---------------|--------------------------------|
| Abundance table        | `otu_table()` | `assay()`, which can hold several (counts, relative abundance, …) |
| Feature information    | `tax_table()` | `rowData()`                    |
| Sample information     | `sample_data()` | `colData()`                  |
| Phylogenetic tree      | `phy_tree()`  | `rowTree()`                    |
| Other data types       | –             | `altExp()`, e.g. the same samples aggregated at another rank |

- **phyloseq** is the original microbiome package. Many older tutorials and packages use it.
- **TreeSummarizedExperiment** builds on Bioconductor's `SummarizedExperiment`, the same   structure used for RNA-seq and other omics data. It can store several assays and   alternative experiments in one object. The *mia* and *miaViz* packages and the   [Orchestrating Microbiome Analysis (OMA)](https://microbiome.github.io/OMA/) book are built around it.

We use TSE in this workshop. If you need phyloseq for another package, you can convert
in both directions (this needs the phyloseq package installed):


``` r
ps  <- convertToPhyloseq(tse)
tse <- convertFromPhyloseq(ps)
```

## Importing the taxonomic profile

nf-core/taxprofiler ran four profilers. Here we use **MetaPhlAn**, because its merged report already contains the full lineage of each taxon (`k__Bacteria|p__…|g__…|s__…`). mia reads this format directly with `importMetaPhlAn()`:


``` r
tse <- importMetaPhlAn(mpa_file)
tse
```

The object has one row per MetaPhlAn species-level genome bin (SGB, `t__…`) and one column per sample. `importMetaPhlAn()` also stored the profile aggregated at every taxonomic rank as alternative experiments (`altExpNames`).


### Tidying sample and taxon names

The column names include the run and database names, and the samples are not in order. We keep only the sample ID and sort the samples:


``` r
colnames(tse)
colnames(tse) <- sub("^(Gut_sample[0-9]+)_.*$", "\\1", colnames(tse))
tse <- tse[, sort(colnames(tse))]
colnames(tse)
```

The taxon names have MetaPhlAn rank prefixes (`g__`, `s__`). We remove them so the plot labels are easier to read:


``` r
for (r in taxonomyRanks(tse)) {
  rowData(tse)[[r]] <- sub("^[a-z]__", "", rowData(tse)[[r]])
}
rowData(tse)[1:3, c("phylum", "genus", "species")]
```

### Exploring the object

Let's look at the three main components:


``` r
assayNames(tse)
assay(tse, "metaphlan")[1:5, ]
rowData(tse)[1:3, c("phylum", "genus", "species")]
taxonomyRanks(tse)
```


MetaPhlAn reports **relative abundance in percent**, not read counts, so each sample sums to about 100:


``` r
colSums(assay(tse, "metaphlan"))
```

::::::::::::::::::::::::::::::::::::::::: callout

### What about sequencing depth?

Because MetaPhlAn reports percentages, we cannot see how many reads each sample had. Use the fastp section of the taxprofiler MultiQC report for read numbers. Profilers that report read counts (Kraken2/Bracken, for example) are compared on Day 2.

::::::::::::::::::::::::::::::::::::::::::::::::

### Adding the sample metadata

We use the samplesheet given to the pipelines as sample metadata. We join it to the sample names of the TSE, so the order of the metadata always matches the order of the columns.

Note the `dplyr::` prefix. mia also has functions called `left_join()`, `inner_join()`, `full_join()` and `right_join()`. Because we loaded mia after dplyr, a plain `left_join()` would call the mia version, which fails on data frames.


``` r
samplesheet <- read_csv(meta_file, show_col_types = FALSE)

col_df <- tibble(sample = colnames(tse)) |>
  dplyr::left_join(select(samplesheet, sample, run_accession, instrument_platform),
            by = "sample")

colData(tse) <- DataFrame(col_df, row.names = col_df$sample)
colData(tse)
```

::::::::::::::::::::::::::::::::::::::::: callout

### Always check that the samples match

If a sample name in the metadata is spelled differently from the one in the abundance table, `left_join()` silently fills that row with `NA`. Check this before continuing:


``` r
stopifnot(!anyNA(colData(tse)$run_accession))
```

::::::::::::::::::::::::::::::::::::::::::::::::

## Aggregating, filtering and transforming

Three operations come before most microbiome analyses:

1. **Aggregate** features at the taxonomic rank of interest. `agglomerateByRank()` sums the    abundances of all rows that share the same genus (or phylum, …).
2. **Filter** features that are rare across samples. `subsetByPrevalent()` keeps taxa that    are above a `detection` threshold in more than a `prevalence` fraction of samples. Rare    taxa add noise and many zeros to downstream statistics.
3. **Transform** the abundances. `transformAssay()` adds a new assay next to the original,    for example relative abundance (proportions between 0 and 1).


``` r
tse_genus <- agglomerateByRank(tse, rank = "genus")
dim(tse_genus)
```

Keep genera with an abundance above 0.1 % in more than half of the samples (at least 3 of 5), then calculate relative abundance:


``` r
tse_genus <- subsetByPrevalent(tse_genus,
                               assay.type = "metaphlan",
                               detection  = 0.1,
                               prevalence = 0.5)
tse_genus <- transformAssay(tse_genus,
                            assay.type = "metaphlan",
                            method     = "relabundance")
dim(tse_genus)
assayNames(tse_genus)
```

::::::::::::::::::::::::::::::::::::: challenge

## Challenge 1: How strict is the filter?

Starting again from `agglomerateByRank(tse, rank = "genus")`, how many genera are kept if a genus must be above 0.1 % in **all five** samples? And in at least **one** sample?

Hint: `subsetByPrevalent()` keeps features whose prevalence is *strictly greater* than `prevalence`. Look at the `include.lowest` argument in `?getPrevalence`.

:::::::::::::::: solution


``` r
tse_g <- agglomerateByRank(tse, rank = "genus")
# All five samples
nrow(subsetByPrevalent(tse_g, assay.type = "metaphlan",
                       detection = 0.1, prevalence = 1,
                       include.lowest = TRUE))
# At least one sample
nrow(subsetByPrevalent(tse_g, assay.type = "metaphlan",
                       detection = 0.1, prevalence = 1/5,
                       include.lowest = TRUE))
```

Without `include.lowest = TRUE`, `prevalence = 1` returns no genera, because a prevalence can never be strictly greater than 100 %. The stricter the prevalence threshold, the fewer taxa remain. No single threshold is correct: it depends on the number of samples and on the question you are asking.

:::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

## Visualising composition

### Composition bar plot

At phylum level a stacked bar plot gives an overview of each sample:


``` r
tse_phylum <- agglomerateByRank(tse, rank = "phylum")
tse_phylum <- transformAssay(tse_phylum, assay.type = "metaphlan",
                             method = "relabundance")
plotAbundance(tse_phylum, assay.type = "relabundance") +
  labs(x = "Sample", y = "Relative abundance") +
  guides(fill = guide_legend(title = "Phylum"),
         colour = guide_legend(title = "Phylum")) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1, vjust = 1))
```

### Prevalence plot

The prevalence plot shows how many genera remain at different detection thresholds (x axis) and prevalence thresholds (one line per prevalence, here 1 to 5 of the 5 samples). It helps when choosing a filter. The detection values are in percent, the units of the `metaphlan` assay:


``` r
tse_g <- agglomerateByRank(tse, rank = "genus")
plotPrevalence(tse_g, assay.type = "metaphlan",
               detection = c(0.01, 0.1, 0.5, 1, 2, 5, 10),
               prevalence = (1:5) / 5, include.lowest = TRUE) +
  scale_x_log10() +
  labs(x = "Detection threshold (% abundance, log scale)", y = "Number of genera")
```

### Heatmap of the most abundant genera

`getTop()` returns the names of the most abundant features. `meltSE()` turns the TSE into a long data frame that ggplot2 can use:


``` r
top_genera <- getTop(tse_genus, top = 15, method = "mean",
                     assay.type = "relabundance")

meltSE(tse_genus[top_genera, ], assay.type = "relabundance") |>
  mutate(FeatureID = factor(FeatureID, levels = rev(top_genera))) |>
  ggplot(aes(x = SampleID, y = FeatureID, fill = relabundance)) +
  geom_tile() +
  scale_fill_viridis_c(name = "Relative\nabundance") +
  labs(x = "Sample", y = "Genus") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

## Importing the functional profile

HUMAnN writes one pathway abundance file per sample (`*_4_pathabundance.tsv.gz`). Each file has two columns: the pathway and its abundance. Some rows are special:

- `UNMAPPED`: reads that did not map to any gene family.
- `UNINTEGRATED`: gene families that could not be assigned to a pathway.
- `PATHWAY|g__Genus.s__species`: stratified rows, which split the pathway's abundance by contributing species.


``` r
path_files <- list.files(humann_dir, pattern = "_4_pathabundance\\.tsv\\.gz$",
                         full.names = TRUE)
basename(path_files)
read_tsv(path_files[1], show_col_types = FALSE, n_max = 5)
```

`read_tsv()` can read all files at once. The `id` argument records which file each row came from. We then keep only the community-level (unstratified) pathways and reshape the table to pathways × samples. Pathways missing from a sample get an abundance of 0:


``` r
path_long <- read_tsv(path_files, id = "file", skip = 1,
                      col_names = c("pathway", "abundance"),
                      show_col_types = FALSE) |>
  mutate(sample = sub("_4_pathabundance\\.tsv\\.gz$", "", basename(file))) |>
  filter(!grepl("|", pathway, fixed = TRUE),
         !pathway %in% c("UNMAPPED", "UNINTEGRATED"))

path_wide <- path_long |>
  select(pathway, sample, abundance) |>
  pivot_wider(names_from = sample, values_from = abundance, values_fill = 0)

path_mat <- as.matrix(path_wide[, -1])
rownames(path_mat) <- path_wide$pathway
path_mat <- path_mat[, colnames(tse)]   # same sample order as the taxonomic TSE
dim(path_mat)
```

Now build a TSE directly with `TreeSummarizedExperiment()`, reusing the sample metadata of the taxonomic object. The HUMAnN abundances are not normalised, so we add relative abundance:


``` r
tse_path <- TreeSummarizedExperiment(
  assays  = list(abundance = path_mat),
  colData = colData(tse)
)
tse_path <- transformAssay(tse_path, assay.type = "abundance",
                           method = "relabundance")
tse_path
```

### Heatmap of the most abundant pathways


``` r
top_paths <- getTop(tse_path, top = 15, method = "mean",
                    assay.type = "relabundance")

meltSE(tse_path[top_paths, ], assay.type = "relabundance") |>
  mutate(FeatureID = factor(FeatureID, levels = rev(top_paths))) |>
  ggplot(aes(x = SampleID, y = FeatureID, fill = relabundance)) +
  geom_tile() +
  scale_fill_viridis_c(name = "Relative\nabundance") +
  labs(x = "Sample", y = "MetaCyc pathway") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

::::::::::::::::::::::::::::::::::::: challenge

## Challenge 2: Which pathways are most abundant?

Which five pathways have the highest mean relative abundance? Are they the same in every sample?

:::::::::::::::: solution


``` r
getTop(tse_path, top = 5, method = "mean", assay.type = "relabundance")

# Top pathway in each sample
apply(assay(tse_path, "relabundance"), 2, function(x) names(which.max(x)))
```

The most abundant pathways are core housekeeping functions (peptidoglycan maturation, the pentose phosphate pathway, nucleotide biosynthesis and glycolysis), which many gut bacteria share. The same pathway is the most abundant in four of the five samples, although their taxonomic compositions differ a lot.

:::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::: instructor

Suggested timing (30 min): libraries and phyloseq vs TSE (5 min), MetaPhlAn import and tidying (7 min), aggregating, filtering and Challenge 1 (7 min), plots (4 min), HUMAnN import, heatmap and Challenge 2 (7 min). If time is short, demonstrate the HUMAnN section and leave Challenge 2 for after the session.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- A TreeSummarizedExperiment keeps the abundance tables (`assay`), feature information (`rowData`) and sample information (`colData`) together. phyloseq objects can be converted to and from it.
- `importMetaPhlAn()` reads a MetaPhlAn merged report, with the full taxonomy, into a TreeSummarizedExperiment.
- HUMAnN pathway tables are read with `readr`, reshaped with `tidyr` and assembled with `TreeSummarizedExperiment()`.
- `agglomerateByRank()`, `subsetByPrevalent()` and `transformAssay()` aggregate, filter and transform the data.
- Check that sample names match between abundance tables and metadata before any analysis.

::::::::::::::::::::::::::::::::::::::::::::::::
