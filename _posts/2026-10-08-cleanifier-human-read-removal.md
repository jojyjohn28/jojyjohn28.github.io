---
layout: post
title: "Cleanifier: Fast Human Read Removal and the False-Positive Question — Day 4 of 5"
date: 2026-10-08
description: "Day 4 of the host-read removal benchmark. Cleanifier finished each sample in under two minutes — roughly 47 times faster than BMTagger — while removing a nearly identical percentage of reads. But a faster tool that removes more reads raises an important question: are those extra removed reads actually human? This post documents a read-level discordance analysis comparing Cleanifier and BMTagger on approximately 194 million read pairs, screening the reads each tool removes exclusively against a nucleotide database to ask whether any confirmed microbial signal is present among them."
comments: true
giscus_comments: true
featured: true
permalink: /blog/human-contamination-removal-cleanifier-false-positives-day4/
series: host-removal-benchmark
order: 4
tags:
  [
    metagenomics,
    host-removal,
    Cleanifier,
    BMTagger,
    false-positives,
    human-contamination,
    benchmarking,
    read-discordance,
    microbial-specificity,
    reproducibility,
    bioinformatics,
    beginners,
  ]
---

🧬 *Day 4 of 5 — Benchmarking Host-Read Removal for High-Host-Content Metagenomes*

The first three days of this benchmark established a clear pattern: BMTagger removed the most reads, Bowtie2 was faster and operationally simpler, and BBMap's output shifted meaningfully depending on which human reference it used. Runtimes ranged from 50 to 77 minutes per sample. Memory requirements peaked at 45 GB for BBMap.

Then I ran Cleanifier. The runtime for Sample 1 — a 41 million read-pair library — came back at 1 minute and 34 seconds.

That number changed the conversation.

A tool that runs in 1.5 minutes while removing roughly the same percentage of reads as a tool that takes 73 minutes is not a minor improvement. It changes what is computationally feasible at scale. But it also raises an important question that a runtime table cannot answer: **if Cleanifier removes slightly more reads than the alignment-based tools, and does so much faster, are those extra removed reads actually human — or are some of them microbial?**

This post documents the read-level analysis I ran to answer that question.

---

## What Cleanifier is

[Cleanifier](https://github.com/smaegol/cleanifier) takes a fundamentally different approach from the alignment tools tested on Days 2 and 3. Rather than aligning reads to a single linear human reference genome, it uses a probabilistic index built from multiple human sequence sources:

- T2T (Telomere-to-Telomere) complete assembly
- Human pangenome sequences
- HLA and variant sequences
- cDNA-related content

This broader representation of human genomic diversity is why Cleanifier can be both faster and potentially more sensitive than a single-reference aligner — it classifies reads probabilistically against a richer picture of human sequence space rather than performing base-by-base alignment against one reference.

---

## Running Cleanifier

The command is clean and straightforward:

```bash
/usr/bin/time -v -o ${SAMPLE}_cleanifier.time.txt \
  cleanifier filter \
    --index human_probabilistic_index \
    --fastq  ${SAMPLE}_R1.fastp.fastq.gz \
    --pairs  ${SAMPLE}_R2.fastp.fastq.gz \
    --prefix ${OUTDIR}/${SAMPLE} \
    --threshold 0.5 \
    --compression gz \
    --compression-threads 16
```

**Key parameters:**

| Parameter | Value | What it does |
|---|---|---|
| `--index` | Probabilistic human index | The pre-built multi-source human index |
| `--threshold 0.5` | Classification cutoff | Reads with probability ≥ 0.5 of being human are removed |
| `--compression gz` | Output format | Writes gzipped FASTQ directly |
| `--compression-threads 16` | Parallel compression | Compresses output while classifying — contributes to the runtime advantage |

The `--threshold` value is worth noting. Lowering it (e.g., 0.3) would remove more reads but increase false-positive risk. Raising it (e.g., 0.7) would be more conservative. For this benchmark, 0.5 was used as the default.

---

## Removal percentages: Cleanifier vs BMTagger

Across the five benchmark samples:

| Sample | BMTagger | Cleanifier | Difference |
|---|---|---|---|
| Sample 1 | 94.88% | 95.73% | +0.85% |
| Sample 2 | 97.91% | 98.88% | +0.97% |
| Sample 3 | 22.51% | 22.72% | +0.21% |
| Sample 4 | 10.52% | 10.61% | +0.09% |
| Sample 5 | 86.07% | 86.83% | +0.76% |
| **Mean** | **62.38%** | **62.95%** | **+0.57%** |

Cleanifier removed slightly more reads than BMTagger in every sample. The difference is largest in the high-host samples (Samples 1, 2, and 5) and negligible in the low-host sample (Sample 4) — a pattern consistent with what was seen when comparing BBMap references on Day 3.

---

## The runtime comparison

| Tool | Mean wall-clock runtime | Approximate peak RAM |
|---|---|---|
| BMTagger | ~73.2 min | ~8.0 GB |
| Bowtie2 | ~50.7 min | ~4.0 GB |
| BBMap GRCh38 | ~77.3 min | ~45.5 GB |
| BBMap masked hg19 | ~55.6 min | ~44.3 GB |
| **Cleanifier** | **~1.56 min** | **~6.6 GB** |

The ~47× runtime advantage over BMTagger is real and reproducible across all five samples. At scale — 100 samples, 200 samples — this is the difference between a preprocessing step that takes a day of cluster time and one that takes under an hour.

The memory requirement (~6.6 GB) is also well within the range of most HPC node configurations, making Cleanifier far easier to schedule than BBMap at scale.

At this point, the obvious summary would be:

> *"Cleanifier removes at least as many human reads as BMTagger while running 47× faster — it is the clear winner."*

But that conclusion skips the question that matters most for downstream biology: are those extra removed reads genuinely human, or does the probabilistic classification approach introduce microbial false positives?

---

## Read-level discordance analysis

To answer this, I reconstructed read-pair-level classifications for BMTagger and Cleanifier across all five samples. Every original fastp-preprocessed read pair was assigned to one of four categories:

| BMTagger decision | Cleanifier decision | Category |
|---|---|---|
| Remove | Remove | **Both remove** — high confidence human |
| Remove | Keep | **BMTagger-only** — BMTagger removes, Cleanifier retains |
| Keep | Remove | **Cleanifier-only** — Cleanifier removes, BMTagger retains |
| Keep | Keep | **Neither removes** — retained by both |

Across approximately 194.5 million total read pairs:

| Category | Read pairs |
|---|---|
| Both remove | 126,193,291 |
| BMTagger-only | 204,189 |
| Cleanifier-only | 1,373,154 |
| Neither removes | 66,717,868 |
| **Total discordant** | **1,577,343** |

**Overall disagreement: 1,577,343 / 194,488,502 ≈ 0.81%**

The two methods agree on more than 99% of read pairs. But the disagreement is strikingly asymmetric: approximately **87% of all discordant pairs are Cleanifier-only removals** — reads that Cleanifier removes but BMTagger retains.

The 1.37 million Cleanifier-only read pairs are the population that demands scrutiny.

---

## What are the Cleanifier-only reads?

The 1,373,154 Cleanifier-only read pairs correspond to 2,746,308 individual sequences. These were extracted and screened against `core_nt` to ask: do any of these reads have credible microbial matches?

The initial screen identified **79,202 sequences** with candidate prokaryotic evidence — about 2.9% of the Cleanifier-only pool.

That initial hit rate sounds concerning. But accepting the top bacterial hit would be too permissive. Human and microbial sequences can produce ambiguous matches for short alignments, conserved regions, low-complexity sequence, and regions where the reference database itself contains contamination. A credible false-positive claim requires competing evidence — the same read must produce stronger evidence for a microbial origin than for a eukaryotic one.

After evaluating bacterial-candidate reads against competing prokaryotic and eukaryotic evidence:

| Classification | Sequences | % of candidates |
|---|---|---|
| Confident bacterial | 4,319 | 5.45% |
| Eukaryotic supported | 59,536 | 75.17% |
| Ambiguous | 13,209 | 16.68% |
| Prokaryotic short/weak | 2,138 | 2.70% |
| **Total** | **79,202** | **100%** |

Three quarters of initial bacterial-looking candidates had better eukaryotic support when competing evidence was considered. That is reassuring — it suggests the initial hits were largely due to conserved or repetitive regions with eukaryotic origins, not genuine microbial contamination of the removed pool.

The 4,319 confident bacterial sequences represent the true false-positive concern.

---

## How large is the confirmed bacterial fraction?

Relative to all Cleanifier-only individual sequences:

```
4,319 / 2,746,308 ≈ 0.157%
```

Numerically, this is very small. Cleanifier is not indiscriminately removing large numbers of microbial reads.

However — and this matters — the biologically important question is not only about percentages. It is about which organisms those reads come from, and whether their removal affects downstream community interpretation.

Among the confirmed bacterial sequences, several reads matched organisms relevant to host-associated microbial communities, including organisms in genera that can be biologically important in the sample type being studied. The strongest single-organism signal within the confirmed bacterial set came from one taxon in particular — appearing across multiple benchmark samples.

931 confirmed sequences mapped to this organism across three samples. As an absolute number, that is tiny relative to 194 million total read pairs. But if this organism is a biologically important community member in your study system — one that you are specifically trying to quantify or compare between groups — its systematic underrepresentation is not a rounding error. It is a bias that compounds across samples.

---

## The reciprocal control: BMTagger-only reads

At this point it would be tempting to declare Cleanifier the tool with the specificity problem and BMTagger the cleaner baseline. But this comparison only holds if BMTagger's exclusive removals are genuinely human.

BMTagger exclusively removed **204,189 read pairs** — reads that Cleanifier retains but BMTagger discards. These 408,378 individual sequences require the same screening framework:

- How many produce confident bacterial hits?
- How many have better eukaryotic support?
- Are any of them from biologically important host-associated organisms?

The reciprocal control is essential before any specificity claim can be made. If BMTagger-only reads also contain a small confirmed microbial population — even a smaller one — then both tools have measurable false-positive rates, and the comparison shifts from "which tool is specific?" to "which tool has an acceptable false-positive rate given its other advantages?"

That analysis is the last piece of the benchmark, and it forms the core of tomorrow's Day 5 summary.

---

## What Day 4 established

Cleanifier's runtime advantage is genuine and large. At ~47× faster than BMTagger with comparable removal rates and a memory footprint that fits comfortably on standard HPC nodes, it changes what is feasible for preprocessing large cohorts.

The false-positive analysis reveals something more nuanced than a simple pass/fail verdict. The vast majority of Cleanifier-only reads have eukaryotic support or are ambiguous — not microbial. A small but confirmed bacterial population does exist among Cleanifier's exclusive removals. Whether that population represents an acceptable tradeoff depends on the study system, the organisms of interest, and the sequencing depth available.

The honest framing is:

> **A ~0.16% confirmed bacterial signal among Cleanifier-exclusive removals may be negligible for most applications — but it is not zero, and it is not evenly distributed across all organisms.**

That distinction is what the Day 5 comparison table will need to make explicit.

---

## What comes tomorrow

Day 5 brings the complete comparison: all five methods side by side, with removal rates, runtime, RAM, read-level agreement, the reciprocal BMTagger-only screen, and a practical recommendation for researchers working with high-host-content metagenomes.

The final question is not "which tool removed the highest percentage?" It is which tool gives the best practical balance across all the dimensions that actually matter for reproducible, biologically interpretable research.

---


![See your plot](/assets/img/cl_hc4.pmg)