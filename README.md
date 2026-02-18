# GATK-Multi-Sample-VCF-Merging
This document describes the process used to merge X individual gVCF samples into a single multi-sample VCF.

## 1. Environment Setup & Preprocessing

### Conda Environment Creation
To ensure compatibility with GATK4 and Samtools, the following environments were created:

```bash
# GATK
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
conda create -n gatk_env gatk4 -y
conda activate gatk_env
```

```bash
# Samtools Environment (for indexing)
conda create -n fasta_index_env -c conda-forge -c bioconda samtools=1.19 -y
```

## 2. Reference Genome Standardization
The input VCFs were identified to use `chr` prefixes. The reference FASTA was modified to match this convention:

### Identify Contig Naming
This step is crucial because bioinformatics tools are extremely "literal." If your reference genome calls a chromosome `chr1` but your VCF calls it `1`, GATK will assume they are completely different organisms and crash.
`zgrep` allows you to look inside a compressed (.gz) VCF file without unzipping it first.

```bash
zgrep "^##contig" [VCF_FILE]
```

Return:
```contig=<ID=chr1,length=248956422,assembly=GRCh38>```

By running the command, you are checking the ID field to see if the files use the "chr" prefix or not.

[!WARNING]

**Check before proceeding:** If your VCF headers and your reference FASTA already use the same naming convention (e.g., both use chr1 or both use 1), skip the renaming and re-indexing steps below and move directly to **Step 3.** Mismatching these will cause GATK to fail.

### Rename Headers
```
awk '/^>/ { if ($1 ~ /^>[0-9XYM]+$/) { sub(/^>/, ">chr") } } {print}' GRCh38.fa > GRCh38.chr.fa
```

### Create Dictionary & Index:
```bash
gatk CreateSequenceDictionary -R GRCh38.chr.fa
conda activate fasta_index_env
samtools faidx GRCh38.chr.fa
conda activate gatk_env
```

## 3. Sample Mapping
A `sample_map.txt` was generated containing multiply entries in the format:
[Sample_ID] [Tab] [Path_to_gVCF]

## 4. GenomicsDB Consolidation

### Prerequisites
* **Samples:** multiple gVCF files (`.g.vcf.gz` + `.tbi` indexes).
* **Reference:** GRCh38 (chromosome naming must be harmonized, since some BED files do not include the `chr` prefix).
* **Intervals:** `targets.bed`.

### GenomicsDB Consolidation
Instead of merging the whole genome (which caused memory crashes and infinite wait times), we used a **Targeted Import** strategy. 

### Execution Command:
```bash
gatk --java-options "-Xmx12g" GenomicsDBImport \
  --genomicsdb-workspace-path pgx_cohort_db \
  --sample-name-map sample_map.txt \
  --intervals targets.bed \
  --interval-padding 1000 \
  --batch-size 50 \
  --bypass-feature-reader true \
  --genomicsdb-shared-posixfs-optimizations true \
  --overwrite-existing-genomicsdb-workspace true
```
