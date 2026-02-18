# GATK-Multi-Sample-VCF-Merging
This document describes the process used to merge X individual gVCF samples into a single multi-sample VCF.

## 1. Environment Setup & Preprocessing

### Conda Environment Creation
To ensure compatibility with GATK4 and Samtools, the following environments were created:

```bash
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
conda create -n gatk_env gatk4 -y
conda activate gatk_env
```

### Samtools Environment (for indexing)
conda create -n fasta_index_env -c conda-forge -c bioconda samtools=1.19 -y


### Reference Genome Standardization
The input VCFs were identified to use `chr` prefixes. The reference FASTA was modified to match this convention:

### Identify Contig Naming
```Verified via zgrep "^##contig" [VCF_FILE]```

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

### Sample Mapping
A `sample_map.txt` was generated containing multiply entries in the format:
[Sample_ID] [Tab] [Path_to_gVCF]

## 2. GenomicsDB Consolidation

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
