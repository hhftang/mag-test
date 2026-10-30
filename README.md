# mag-test

A test of nf-core/mag functionality and overview of a short-read metagenome pipeline.

## Overview

This repository contains the output from a test run of nf-core/mag v5.5.0 (https://nf-co.re/mag/5.5.0/), a community best-practice pipeline for the assembly, binning and annotation of metagenomes. The run was performed within a Docker container.

Due to storage limitations, it does not contain all the files required for and produced by the run. Instead, it includes:

- Basic execution reports of the run in `test-output/pipeline_info/`
- Multiqc report of the run in `test-output/multiqc/`
- Taxonomy information in `test-output/taxonomy/`
- Selected Nextflow config files from nf-core/mag in `conf/`.
- Selected Nextflow script files from nf-core/mag in `mag-copy/`. These contain scripts that run the analysses, nested within `/workflows/` and `/subworkflows/`. 

This readme text contains my own notes on what each tool and each part of the pipeline does. This was done by manually parsing through and reading the `.command.sh` output for each process, then consulting the documentation.

## Details

The test run configuration is derived from test.config, and described in `config_log.log`.

The test run execution reports are given in `execution_log.log` and the various .html files in `test-output/pipeline_info/`. The latter indicate resource use and can be used to guide future resource allocation.

### Input

- Input is the **samplesheet** `sample.multirun.v4.csv`, a file that points to short-read fastq files for samples of the `test_minigut` dataset. This is a minimal and synthetic "minigut" dataset from nf-core/test-datasets, generated from a single genome of Bacteroides fragilis YCH46 (https://github.com/nf-core/test-datasets/tree/mag). It is commonly used for test runs like this one.

### Pre-processing

- **FastQC:** applied to both raw and trimmed reads. Common quality control tool that reports sequence quality.
- **fastp:** used for adapter trimming. Fast tool used for quality control, dapater trimming and preprocessing. Quality value set at 15 (default). Sliding window `--cut_front` and `--cut_tail` options used, quality requirement set at 15 (default 20). Min length required set at 15 (default).
- **bowtie2-build:** used to build a BWT/FM index from a reference genome to allow rapid alignment. This was applied to `GCA_002596845.1_ASM259684v1_genomic.fna.gz` (default), a reference genome supplied in nf-core/test-datasets and typically used to remove Illumina PhiX contaminant reads.
- **bowtie2:** used for PhiX removal, aligning to the above reference. Fast tool used for DNA/RNA alignment to long reference sequences.
- **Runs were merged** with `cat`
- The input had **no long-read sequences**. Therefore, all standard long-read steps (Nanoplot, porechop-abi, chopper) were skipped.

### Assembly

- **SPAdes/metaSPAdes:** applied to merged runs. Tool used to perform de novo genome assembly from sequencing data. metaSPAdes is used for complex multi-species metagenomic datasets with uneven abundances and high diversity. Uses multi-sized de-Bruijn graphs, bulge projection, and the exSPAnder repeat-resolution algorithm.
- **MEGAHIT:** applied to merged runs. Fast tool used for NGS de novo assembly. Uses succinct de Bruijn graphs (SdBG) and an iterative multiple k-mer size strategy to lower memory usage and process massive datasets quickly without heavy pre-processing.
- **ALE:** skipped because the test reads are not properly paired (according to test.config comment, although technically it can also be used on unpaired reads). Reference-independent framework used to assess and score accuracy of genome and metagenome assemblies. Identifies single-base errors, indels, rearrangements, and chimeras. Uses a Bayesian statistical framework combining: K-mer frequency prior, read placement and alignment quality, insert length distribution, and sequencing depth and coverage.
- **QUAST/MetaQUAST:** applied to gunzipped contigs. Evaluation tool for assessing quality of contigs against known reference genomes. Prediction for rRNA identification enabled (`--rna-finding`).
- Hybrid, Flye, and metaMDBG were skipped (no long reads)

### Gene prediction

- **Prodigal:** ran on assembled contigs. Fast microbial gene prediction tool that supports metagenomic and fragmented sequence data. Unlike normal mode which requires a longer sequence or trains on the input genome itself, the `-p meta` flag bypasses self-training on individual contigs and uses a pre-accumulated or generalized heuristic model ideal for short, noisy, or heterogeneous metagenomic assemblies and plasmids.

### Binning

- **bowtie2-build and bowtie2:** used to prepare for binning. Specifically, aligning QCed reads back to assembled contigs.
- **MetaBat2:** used for binning. Tool for automated adaptive binning, reconstructing individual microbial genomes from metagenomic assembly. `jgi_summarize_bam_contig_depths` was first applied to calculate depth of coverage for contigs across one or more sorted BAM files, and prepare for binning. A custom bioawk script then converts the output into a tabular format that can be read by subsequent binning tools (as abundance files). `Metabat2` was then run with minimum binned contig length 1500 (default 2500).
- **MaxBin2:** used for binning. Tool that clusters assembled metagenomic contigs into individual microbial genome bins using an EM algorithm.
- **SemiBin:** used for binning. Tool that uses self-supervised contrastive learning and Siamese neural networks to group assembled contigs into discrete genomic bins. Single-sample binning (`single_easy_bin`) was used for this, with minimum length for binned contigs set at 1500 (default 1000 or 2500).
- **seqkit:** used to generate statistics of binned files.
- **Unbinned contig thresholds** were relaxed for testing: Minimum length of unbinned contigs is set to 1 (default 1000,000), and max number of unbinned contigs set to 2 (default 100). This appears to be implemented using `split_fasta.py`.
- Other scripts in nf-core/mag (`get_mag_depths.py`, `get_mag_depths_summary.py`) retrieved information on sequencing depth or coverage and presented it as a summary table.
- **CONCOCT**, **COMEBin**, and **MetaBinner** were skipped to save time for this test run.

### Bin QC

- **BUSCO:** Tool (Benchmarking Universal Single-Copy Orthologs) used to evaluate the quantitative completeness and redundancy of genome assemblies, including metagenomes, using Hidden Markov Models. It measures how well a sequence assembly captures expected evolutionarily conserved, single-copy ortholog genes. The BUSCO lineage dataset was set to one stored within nf-core/test-datasets `bacteria_odb10.2024-01-08.tar.gz`. This is a domain-level lineage dataset with 124 conserved single-copy orthologs. `qsv` is then used concatenate the BUSCO output.
- The test.config file activates an option (`busco_clean`) to tidy up temporary files after BUSCO runs.
- **QUAST:** called once again, this time to QC each binned contig. Both `--rna-finding` and `--gene-finding` are enabled. Again, `qsv` is called to concatenate the QUAST output.

### Taxonomic profiling

- **CAT_pack:** CAT (Contig Annotational Tool) is a pipeline used for taxonomic classification of assembled metagenomic contigs. It predicts ORFs using Prodigal, queries protein translation against reference databases via DIAMOND, and applies a voting-based last common ancestor (LCA) approach to classify contigs. 
    - In this run it used the minigut test database `minigut_cat.tar.gz`, from nf-core/test-datasets. This is a downsized version of the original CAT (Contig Annotational Tool) taxonomic database. 
    - `CAT_pack bins` classifies each bin taxonomically by searching from protein sequences against NCBI reference database using DIAMOND and LCA. It outputs a bin-to-taxonomy classification file.`test.config` disabled the default setting of CAT to mark the most probable hits with an asterix in output files.
    - `CAT_pack add_names` annotates each bin with its full, human-readable taxonomy lineage, filtering out ambiguous names.
    - `CAT_pack summarise` generates a taxonomic summary report from the full bin-to-taxonomy classification file. It collapses the lineage into a single best taxonomic assignment per bin (typically at genus or species level).
    - The same functions are then applied to **classify unbinned contigs**.
- **GTDB-Tk:** Tool used to assign objective taxonomic classifications to bacterial (and archaeal) genomes using the Genome Taxonomy Database (GTDB).
    - In this run, it used a mock-up GTDB database (r232, May 2026), again part of nf-core/test-datasets.
    - `gtdbtk classify_wf` was used to identify marker genes, build alignment, and place genomes on the GTDB reference tree. Minimum percentage of amino acids required for a gene to be considered was set to 10% (default), and minimum alignment fraction set at 65% (default 50%). The tool was set to use the full bacterial tree and place species labels.
    - Minimum completeness threshold was lowered to 1% (default 50%), for the purposes of the test run.
    - `summary_gtdbtk.py` was used to merge the GTDB-Tk classification summaries from multiple MAG binners (MetaBAT2, MaxBin2, SemiBin2) into a single unified table.

### Annotation and summarisation

- `combine_tables.py` merges 5 per-MAG metric tables - assembly quality (QUAST), read coverage (depths), GTDB taxonomy (GTDB-Tk), NCBI taxonomy (CAT), and gene completeness (BUSCO) - into a single bin_summary.tsv file. This is the final consolidated summary of all MAGs produced by the pipeline, one row per bin with all quality and annotation columns combined.
- **PROKKA:** A rapid tool for annotating prokaryotic genomes and metagenomic contigs. Uses Prodigal for protein-coding gene (CDS) prediction, Aragorn for tRNAs, Barrnap/RNAmmer for rRNAs, Infernal for non-coding RNAs, and BLAST+/HMMER3 for functional annotation. It was run in fast mode for this run.
- **MultiQC:** Tool for aggregating and visualising quality control results from multiple bioinformatic tools across a pipeline.

## Details on Multiqc report

- A mock GTDB-Tk database was used for the test run to facilitate speed of testing. The taxonomical results given are therefore not based in biological reality. (see https://data.ace.uq.edu.au/public/gtdb/data/releases/latest/auxillary_files/gtdbtk_package/mockup_db/HOWTO.txt)
