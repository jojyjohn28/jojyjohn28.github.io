---
layout: post
title: "Sequali: A Modern Alternative to FastQC for Sequencing Quality Control"
date: 2026-10-20
description: "A practical introduction to Sequali, a fast and lightweight sequencing quality-control tool for Illumina and Nanopore data. Learn about its quality metrics, adapter detection, duplication estimation, MultiQC integration, and how it compares with FastQC."
comments: true
giscus_comments: true
featured: true
categories: [genomics]
permalink: /blog/sequali-sequencing-quality-control-fastqc-alternative/
tags: [bioinformatics, sequencing, quality-control, FASTQ, Sequali, FastQC, Illumina, Nanopore, MultiQC, HPC, beginners]
---

🧬 **Day 105 of Daily Bioinformatics from Jojy's Desk**

## Sequali: A Modern Alternative to FastQC for Sequencing Quality Control

Quality control is one of the first steps in almost every sequencing workflow. Whether we are analyzing whole genomes, metagenomes, metatranscriptomes, or amplicons, we start by asking similar questions:

- Are the reads of acceptable quality?
- Is quality dropping toward the ends of the reads?
- Are adapters still present?
- Are there overrepresented sequences?
- Is the library highly duplicated?
- Does the read-length distribution look reasonable?

For years, **FastQC** has been the default tool for answering these questions. Recently I came across **Sequali**, which offers some interesting improvements, particularly for people working with both short-read and long-read data. Today: what Sequali does, how to run it, and how it compares with FastQC.

---

## 1. What is Sequali?

[Sequali](https://github.com/rhpvorderman/sequali) is an open-source sequencing QC tool developed by Ruben H. P. Vorderman and described in a 2025 paper in *Bioinformatics Advances*: **"Sequali: efficient and comprehensive quality control of short- and long-read sequencing data."**

Instead of targeting a single platform, Sequali provides one QC framework for both Illumina and Oxford Nanopore data. It reads FASTQ files and unaligned BAM (uBAM) files.

Notable features:

- Fast execution with low memory requirements
- Quality-score and nucleotide-composition plots
- Adapter detection for Illumina and Nanopore reads
- Overrepresented sequence identification
- Duplication-rate estimation
- Insert-size analysis for paired-end reads
- Illumina per-tile quality plots
- Nanopore-specific channel and translocation-speed plots
- MultiQC integration

Sequali writes an HTML report for visual inspection and a JSON file for downstream processing.

---

## 2. Why is Sequali interesting?

At first glance it looks like another FastQC-style tool. Three implementation details are worth a closer look.

### A. Read-level quality is averaged as error probability

Phred scores are logarithmic:

```text
Q = -10 × log10(P)
```

where P is the probability of an incorrect base call.

| Phred score | Estimated error probability |
|---|---|
| Q10 | 10% |
| Q20 | 1% |
| Q30 | 0.1% |
| Q40 | 0.01% |

Because the scale is logarithmic, averaging Phred scores directly is not the same as averaging the underlying error. Consider a read with just two bases, Q10 and Q30:

- Arithmetic mean of the scores: **Q20**
- Mean error probability: (0.1 + 0.001) / 2 = 0.0505, which converts back to roughly **Q13**

The naive average hides the fact that the poor base dominates the expected number of errors. Sequali summarizes read-level quality using expected error probabilities: convert each base to an error probability, average, then convert back to Phred.

### B. Overrepresented sequences, with an attempt at identification

Overrepresented sequences can point to adapter contamination, PCR enrichment, unexpected library-prep sequences, or other technical artifacts. According to the paper, Sequali uses short sequence fragments (21 bp) to find frequent patterns and compares candidates against the NCBI UniVec database and additional known sequencing-related sequences, so you get a hint about *what* an overrepresented sequence might be, not only that it exists.

A caution: an overrepresented sequence is not automatically contamination. Genuine biological sequences can also be highly abundant.

### C. Duplication estimation by fingerprint subsampling

Duplication can come from PCR amplification, low library complexity, or genuine biological abundance. Sequali estimates it using a fingerprint-based subsampling approach: instead of storing every complete read, it samples short fingerprints and uses their frequency distribution to estimate duplication. This keeps memory use down while still producing a duplication profile.

**A caution for microbiome researchers:** high duplication does not necessarily mean poor data. In metagenomes and metatranscriptomes, abundant organisms and highly expressed transcripts naturally generate repeated sequences. Always interpret duplication in the context of the experiment.

---

## 3. Sequali vs. FastQC

FastQC remains a useful, established tool. Sequali builds on many of the same ideas and adds some capabilities.

| Feature | FastQC | Sequali |
|---|---|---|
| Illumina FASTQ QC | Yes | Yes |
| Nanopore-specific QC | Limited | Yes |
| Per-base quality plots | Yes | Yes |
| Per-read quality summaries | Yes | Yes (expected-error approach) |
| GC-content analysis | Yes | Yes |
| Adapter detection | Yes | Yes, including ONT adapters |
| Overrepresented sequences | Yes | Yes, with expanded identification |
| Duplication analysis | Yes | Yes (fingerprint-based estimation) |
| Paired-end insert-size metrics | No native module | Yes |
| Illumina per-tile quality | Yes | Yes |
| Nanopore channel plots | No | Yes |
| MultiQC integration | Yes | Yes |
| HTML reports | Yes | Yes |

### What about speed?

The Sequali paper benchmarked it against other QC tools. The numbers below are as reported in the publication, **not from my own benchmark**.

Illumina benchmark dataset:

| Tool | Runtime |
|---|---|
| Sequali | 19 seconds |
| Falco | 35 seconds |
| fastp | 26 seconds (with adjustments described in the paper) |
| NanoPlot | 2 min 41 s |

Nanopore benchmark dataset:

| Tool | Runtime |
|---|---|
| Sequali | 3 min 22 s |
| Falco | 17 min 12 s |
| fastp | 8 min 39 s (with adjustments described in the paper) |
| NanoPlot | 23 min 28 s |

Note that the study compared against **Falco**, a faster FastQC-compatible reimplementation, not the original FastQC executable. The results suggest Sequali can be substantially faster, especially on long reads, but actual performance depends on dataset size, read length, compression, hardware, and software versions. Run it on your own data before drawing conclusions.

---

## 4. Installing Sequali

Sequali is available through pip and Bioconda.

**Option 1: pip**

```bash
pip install sequali
```

**Option 2: conda (my preference, in its own environment)**

```bash
conda create -n sequali_env -c conda-forge -c bioconda sequali
conda activate sequali_env
```

Check the installation:

```bash
sequali --help
```

---

## 5. Running Sequali on FASTQ files

### Single-end reads

```bash
sequali sample.fastq.gz
```

By default this writes two reports next to the input:

```text
sample.fastq.gz.html
sample.fastq.gz.json
```

Open the HTML in a browser; the JSON holds the same QC information in a structured form for scripts.

### Paired-end Illumina reads

```bash
sequali sample_R1.fastq.gz sample_R2.fastq.gz
```

With paired input Sequali can look at read overlap and insert-size characteristics, which is a useful quality indicator for Illumina libraries.

### Organizing output

For many samples, keep reports in a dedicated directory:

```bash
mkdir -p 02_qc/sequali

sequali \
  --out-dir 02_qc/sequali \
  sample_R1.fastq.gz \
  sample_R2.fastq.gz
```

Note the flag is `--out-dir` (with a hyphen). Option names can change between versions, so confirm against `sequali --help` for the version you installed.

---

## 6. Before-and-after quality control

A practical use is comparing quality before and after preprocessing. Suppose you use fastp to trim adapters and filter low-quality reads.

Assess the raw reads:

```bash
sequali \
  --html before_qc.html \
  --json before_qc.json \
  raw.fastq.gz
```

Preprocess:

```bash
fastp \
  -i raw.fastq.gz \
  -o trimmed.fastq.gz \
  -h fastp.html \
  -j fastp.json
```

Assess the processed reads:

```bash
sequali \
  --html after_qc.html \
  --json after_qc.json \
  trimmed.fastq.gz
```

Compare the two reports for changes in base-quality distributions, adapter content, read lengths, overrepresented sequences, and duplication profiles.

One distinction matters: **Sequali assesses quality; fastp preprocesses reads.** They are complementary, not replacements for each other.

---

## 7. Integrating Sequali with MultiQC

With hundreds of samples, opening individual HTML reports is impractical. MultiQC can aggregate Sequali results; support was added in MultiQC version 1.22.

```bash
conda install -c conda-forge -c bioconda multiqc

multiqc 02_qc/sequali -o 02_qc/multiqc
```

This produces a single project-level report, which is especially handy for large metagenomic or transcriptomic cohorts. Make sure your MultiQC is version 1.22 or newer.

---

## 8. Using Sequali on an HPC cluster

Sequali's low memory footprint suits shared computing environments. Its documentation indicates typical analyses need only a couple of GB of RAM, and by default it uses about two CPU cores, though a single-core mode is available (see `sequali --help` for the option in your version). Resource use depends on the input, so measure rather than assume.

Because each sample is independent, many single-sample jobs can be more efficient than giving extra cores to any one sample. A simple SLURM script:

```bash
#!/bin/bash
#SBATCH --job-name=sequali_qc
#SBATCH --cpus-per-task=2
#SBATCH --mem=4G
#SBATCH --time=01:00:00
#SBATCH --output=sequali_%j.log

source /path/to/conda.sh        # replace with the correct path on your cluster
conda activate sequali_env

sequali --out-dir 02_qc/sequali sample.fastq.gz
```

For large projects, extend this to a SLURM job array with one sample per task (the same pattern used in the Kaiju and genome-annotation posts in this series). Adjust the memory and time requests after checking actual usage with `sacct` or `/usr/bin/time -v`.

---

## 9. Sequali for Nanopore sequencing

Nanopore data behaves differently from Illumina:

- Read lengths vary enormously
- Adapter and barcode contamination can remain after basecalling
- Quality can vary between channels
- Translocation speed can affect run performance

Sequali provides plots designed for these characteristics, including channel and translocation-speed views, and accepts BAM files from Nanopore basecalling workflows.

```bash
sequali nanopore_reads.fastq.gz
sequali dorado_basecalled.bam
```

One limitation: Sequali does not currently use paired-read relationships in BAM input. For Nanopore data, where reads are analyzed individually, this rarely matters.

---

## 10. Should we replace FastQC with Sequali?

I would not replace an established QC pipeline without evaluating the alternative first. FastQC has a long track record, extensive documentation, and is built into countless workflows.

Sequali's attractions:

- One approach to short-read and long-read QC
- Efficient performance
- Expected-error-based quality summaries
- Additional adapter and overrepresentation analysis
- Paired-end insert-size information
- Low memory requirements

If you are building a new sequencing QC workflow, Sequali is worth evaluating. If you already have one, run FastQC and Sequali on the same representative samples and compare the reports. The interesting question is whether Sequali reveals problems that are not obvious in the FastQC output.

A simple evaluation set:

1. One high-quality Illumina sample
2. One sample with known adapter contamination
3. One highly duplicated library
4. One Nanopore dataset

That would show quickly whether Sequali offers practical advantages for the data you actually analyze.

---

## 11. Final thoughts

QC is sometimes treated as routine: run FastQC, glance at the report, move on. But how a tool calculates and presents its metrics shapes how we interpret the data. Sequali shows that even familiar measurements, like average read quality and duplication, can benefit from different computational approaches, and its support for both Illumina and Nanopore is a real advantage as workflows increasingly combine short and long reads.

The lesson I take from it: **quality control is not simply about generating plots. It is about understanding what those plots measure and whether they accurately describe the sequencing data.**

Sequali is a useful tool to have in the toolkit.

---

## References and resources

1. Vorderman, R. H. P. (2025). Sequali: efficient and comprehensive quality control of short- and long-read sequencing data. *Bioinformatics Advances*, 5(1), vbaf010. https://doi.org/10.1093/bioadv/vbaf010
2. Sequali documentation: https://sequali.readthedocs.io/
3. Sequali GitHub repository: https://github.com/rhpvorderman/sequali
4. FastQC: https://www.bioinformatics.babraham.ac.uk/projects/fastqc/
5. MultiQC: https://multiqc.info/

---

*Day 105 of Daily Bioinformatics from Jojy's Desk.*

*Small steps. Real progress.*