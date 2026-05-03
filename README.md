# WGBS-and-Epigenetic-Aging-Clock-Benchmarking-with-Galaxy-and-Biolearn
The repository presents two complementary epigenetics analyses: WGBS processing using the Galaxy methylation-seq tutorial and a Python-based Biolearn benchmark comparing multiple DNA methylation aging clocks across two complete datasets using visual and quantitative performance evaluation.

# DNA Methylation Data Analysis (WGBS) — Galaxy Tutorial

> Step-by-step guide based on [GTN: DNA Methylation data analysis](https://training.galaxyproject.org/training-material/topics/epigenetics/tutorials/methylation-seq/tutorial.html)  
> **Reference:** Lin et al. 2015 | **Genome:** hg38

---

## 📋 Overview

Whole Genome Bisulfite Sequencing (WGBS) pipeline on Galaxy covering:

1. [Data Upload](#1-data-upload)
2. [Quality Control — Falco](#2-quality-control-falco)
3. [Alignment — bwameth](#3-alignment-bwameth)
4. [Methylation Bias — MethylDackel](#4-methylation-bias-methyldackel)
5. [Methylation Extraction — MethylDackel](#5-methylation-extraction-methyldackel)
6. [Visualization — computeMatrix & plotProfile](#6-visualization)
7. [DMR Detection — Metilene](#7-metilene)

**Samples:** Normal breast (NB), fibroadenoma (BT089), invasive ductal carcinomas (BT126, BT198), adenocarcinoma cell line (MCF7)

---

## Prerequisites

- Galaxy account (e.g. [usegalaxy.org](https://usegalaxy.org))
- Basic familiarity with Galaxy interface
- Reviewed: [Quality Control tutorial](https://training.galaxyproject.org/training-material/topics/sequence-analysis/tutorials/quality-control/tutorial.html) & [Mapping tutorial](https://training.galaxyproject.org/training-material/topics/sequence-analysis/tutorials/mapping/tutorial.html)

---

## 1. Data Upload

**Goal:** Load subset FASTQ files into Galaxy history.

### Steps

1. Create new history in Galaxy (`+` icon in history panel)
2. Click **Upload** → **Paste/Fetch Data**
3. Paste URLs and press **Start**:

```
https://zenodo.org/record/557099/files/subset_1.fastq
https://zenodo.org/record/557099/files/subset_2.fastq
```

4. Close upload window after completion

> **Alternative:** Import from shared data library → `GTN - Material → Epigenetics → DNA Methylation data analysis`

---

## 2. Quality Control (Falco)

**Tool:** `Falco` v1.2.4+galaxy0  
**Goal:** Assess read quality; understand bisulfite-specific base distribution.

### Steps

1. Search and open **Falco** in Galaxy tool panel
2. Set parameters:
   - **Raw read data:** select both `subset_1.fastq` and `subset_2.fastq` (multi-select with `Ctrl`)
3. Run tool and inspect **Per base sequence content** in HTML report

### Expected Output

Unusual T/C distribution due to bisulfite conversion — all unmethylated C → T.  
GC content will appear skewed. **This is expected.**

> Bisulfite converts unmethylated cytosines to uracil (read as T). Methylated C remains C. Normal QC tools flag this as an error — it is not.

---

## 3. Alignment (bwameth)

**Tool:** `bwameth` v0.2.7+galaxy0  
**Goal:** Map bisulfite reads to reference genome using a methylation-aware aligner.

### Steps

1. Open **bwameth** in tool panel
2. Set parameters:

| Parameter | Value |
|-----------|-------|
| Reference genome source | `Use a built-in index` |
| Reference genome | `Human (hg38full)` |
| Library type | `Paired-end` |
| First read in pair | `subset_1.fastq` |
| Second read in pair | `subset_2.fastq` |

3. Run tool

> **Long runtime?** Skip by importing precomputed BAM:
> ```
> https://zenodo.org/records/557099/files/aligned_subset.bam
> ```

> **Why not BWA/Bowtie?** Standard aligners cannot handle C→T conversions. `bwameth` aligns against a 3-letter genome where C is treated as T, then restores proper mapping.

---

## 4. Methylation Bias (MethylDackel)

**Tool:** `MethylDackel` v0.5.2+galaxy0  
**Goal:** Detect position-dependent methylation bias along reads.

### Steps

1. Open **MethylDackel** in tool panel
2. Set parameters:

| Parameter | Value |
|-----------|-------|
| Load reference genome from | `Local cache` |
| Reference genome | `Human (hg38)` |
| Sorted BAM file | output of **bwameth** |
| What do you want to do? | `Determine the position-dependent methylation bias (mbias)` |
| **Advanced options** → Keep singletons | `Yes` |
| **Advanced options** → Keep discordant alignments | `Yes` |

3. Inspect SVG diagnostic plots output

### Interpreting Bias Plot

- Roughly uniform methylation across read positions = acceptable
- Spikes at read start/end → consider trimming
- ±5% variation is normal; trim only if clearly biased
- Example trim: strand 1 → positions 0–145; strand 2 → positions 6–149

---

## 5. Methylation Extraction (MethylDackel)

**Tool:** `MethylDackel` v0.5.2+galaxy0  
**Goal:** Extract CpG methylation fractions from aligned BAM.

### Steps

1. Open **MethylDackel** again
2. Set parameters:

| Parameter | Value |
|-----------|-------|
| Load reference genome from | `Local cache` |
| Reference genome | `Human (hg38)` |
| Sorted BAM file | output of **bwameth** |
| What do you want to do? | `Extract methylation metrics (extract)` |
| Merge per-Cytosine metrics | `Yes` |
| Output options | `CpG methylation fractions (--fraction)` |

3. Output: BedGraph file with per-CpG methylation fractions

---

## 6. Visualization

**Tools:** `Wig/BedGraph-to-bigWig`, `computeMatrix` v3.5.4, `plotProfile` v3.5.4  
**Goal:** Plot methylation levels around CpG islands.

### Part A — Your Extracted Data

1. **Convert to bigWig:**
   - Tool: **Wig/BedGraph-to-bigWig**
   - Input: `fraction CpG` (MethylDackel extract output)
   - If genome not set: edit dataset → set `Database/Build` to `hg38`

2. **Import CpG islands BED:**
   ```
   https://zenodo.org/records/557099/files/CpGIslands.bed
   ```

3. **computeMatrix:**

| Parameter | Value |
|-----------|-------|
| Regions to plot | `CpGIslands.bed` |
| Sample order matters | `No` |
| Score file | output of **Wig/BedGraph-to-bigWig** |
| Output options | `reference-point` |

4. **plotProfile:**
   - Matrix file: output of **computeMatrix**

### Part B — Multi-Sample Precomputed Data

**Chromosome naming issue:** Ensembl uses `1`, UCSC uses `chr1`. Convert using:

```
https://raw.githubusercontent.com/dpryan79/ChromosomeMappings/master/GRCh38_ensembl2UCSC.txt
```

Use **Replace column** tool (v0.2): replace column 1 of bedGraph using mapping file.

**Import preconverted UCSC files** (save time):

```
https://zenodo.org/records/557099/files/NB1_CpG.meth_ucsc.bedGraph
https://zenodo.org/records/557099/files/NB2_CpG.meth_ucsc.bedGraph
https://zenodo.org/records/557099/files/BT089_CpG.meth_ucsc.bedGraph
https://zenodo.org/records/557099/files/BT126_CpG.meth_ucsc.bedGraph
https://zenodo.org/records/557099/files/BT198_CpG.meth_ucsc.bedGraph
https://zenodo.org/records/557099/files/MCF7_CpG.meth_ucsc.bedgraph
```

**Build collection:**
1. Select all 6 files → **Build List**
2. Rename: strip file extensions (e.g. `NB1_CpG.meth_ucsc.bedGraph` → `NB1_CpG`)
3. Label collection: `all_coverage_files`
4. Set datatype to `bedgraph` and database to `hg38` for all files

**Run pipeline on collection:**

```
Wig/BedGraph-to-bigWig (collection input)
    ↓
computeMatrix (reference-point, CpGIslands.bed)
    ↓
plotProfile (Show advanced options: Yes → Make one plot per group: Yes)
```

### Expected Output

Methylation dip at CpG island centers — consistent with promoter silencing pattern across samples.

---

## 7. Metilene

**Tool:** `Metilene` v0.2.6.1  
**Goal:** Detect differentially methylated regions (DMRs) between normal and tumor samples.

### Steps

1. Import bedGraph files from Zenodo:

```
https://zenodo.org/records/557099/files/NB1_CpG.meth.bedGraph
https://zenodo.org/records/557099/files/NB2_CpG.meth.bedGraph
https://zenodo.org/records/557099/files/BT198_CpG.meth.bedGraph
```

2. Open **Metilene** and set parameters:

| Parameter | Value |
|-----------|-------|
| Input group 1 | `NB1_CpG.meth.bedGraph` + `NB2_CpG.meth.bedGraph` |
| Input group 2 | `BT198_CpG.meth.bedGraph` |
| BED file (regions of interest) | `CpGIslands.bed` |

3. Run tool and inspect PDF output

### Output Plots (PDF)

| Plot | Description |
|------|-------------|
| DMR differences distribution | Magnitude of methylation changes |
| DMR length (bp) | Size distribution of DMRs |
| Number of CpGs per DMR | CpG density in DMRs |
| DMR diff vs. q-values | Statistical significance |
| Mean methylation G1 vs G2 | Group comparison scatter |
| DMR length (bp) vs length (CpGs) | Correlation of DMR metrics |

---

## 📊 Workflow Summary

```
FASTQ reads (subset_1, subset_2)
    ↓
Falco (QC)
    ↓
bwameth (Alignment, hg38)
    ↓
MethylDackel mbias (Bias check)
    ↓
MethylDackel extract (CpG fractions → BedGraph)
    ↓
Wig/BedGraph-to-bigWig → computeMatrix → plotProfile (Visualization)
    ↓
Metilene (DMR detection)
```

---

## Data Sources

All data: [Zenodo record 557099](https://zenodo.org/record/557099)

---

## Key Concepts

| Concept | Explanation |
|---------|-------------|
| Bisulfite conversion | Unmethylated C → T; methylated C stays C |
| Why special aligner? | Standard aligners fail on C→T converted reads |
| Methylation bias | Position-dependent artifacts at read ends |
| DMR | Region with significantly different methylation between conditions |
| CpG island | CpG-rich genomic region, often at gene promoters |
| Chromosome naming | Ensembl (`1`) vs UCSC (`chr1`) — must match reference |

---

## Reference

Lin, I.-H. et al. (2015). **Hierarchical Clustering of Breast Cancer Methylomes Revealed Differentially Methylated and Expressed Breast Cancer Genes.** *PLOS ONE* 10: e0118453. [doi:10.1371/journal.pone.0118453](https://doi.org/10.1371/journal.pone.0118453)

**Tutorial:** Wolff, Ryan, Moosmann — [GTN DNA Methylation data analysis](https://training.galaxyproject.org/training-material/topics/epigenetics/tutorials/methylation-seq/tutorial.html)
