# GATK-Multi-Sample-VCF-Merging
This document describes the process used to merge multiple individual gVCF samples into a single multi-sample VCF.

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

OR

```contig=<ID=1,length=248956422,assembly=GRCh38>```


By running the command, you are checking the ID field to see if the files use the "chr" prefix or not.

> [!IMPORTANT]  
> **Check before proceeding:** If your VCF headers and your reference FASTA already use the same naming convention (e.g., both use chr1 or both use 1), skip the renaming and re-indexing steps below and move directly to **Step 3.** Mismatching these will cause GATK to fail.

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

## 3. Sample Map Generation Logic
To facilitate the batch import of multiple samples into GenomicsDB, a bash script was utilized to automate the creation of the `sample_map.txt`. The script iterates through the source directory, utilizing the `basename` command to extract **unique Sample IDs** from the filenames while stripping the `.g.vcf.gz` extensions. To ensure the manifest remains portable and robust against directory changes, the `realpath` command was used to resolve absolute paths for every gVCF file. Crucially, the script enforces the GATK-required **Tab-separated** format between the Sample ID and the File Path.

```bash
#!/bin/bash

GVCF_DIR="/DIRECTORY"
OUTPUT_FILE="sample_map.txt"

echo "Creating sample map file..."
echo "Reading from: $GVCF_DIR"
echo "Writing to: $(pwd)/$OUTPUT_FILE"

> "$OUTPUT_FILE"

for f in "$GVCF_DIR"/*.g.vcf.gz; do
    if [[ -f "$f" ]]; then
        
        # Extract sample name from filename
        sample=$(basename "$f" .g.vcf.gz)
        
        fullpath=$(realpath "$f")
        
        echo -e "${sample}\t${fullpath}" >> "$OUTPUT_FILE"
        
        echo "Added: $sample"
    fi
done

echo "Done."
echo "Sample map saved to:"
echo "$(pwd)/$OUTPUT_FILE"
```

A `sample_map.txt` was generated containing multiply entries in the format:
[Sample_ID] [Tab] [Path_to_gVCF]

## 4. GenomicsDB Consolidation

### Prerequisites
* **Samples:** multiple gVCF files (`.g.vcf.gz` + `.tbi` indexes).
* **Reference:** GRCh38 (chromosome naming must be harmonized, since some BED files do not include the `chr` prefix).
* **Intervals:** `targets.bed`.

Instead of merging the whole genome (which caused memory crashes and infinite wait times), we used a **Targeted Import** strategy. 

### Execution Command:

**4.1 GenomicsDBImport**

The `GenomicsDBImport` tool is designed to scale joint genotyping by organizing individual gVCFs into a specialized 2D columnar data structure (TileDB). The parameters used in this workflow were specifically chosen to optimize performance for a X-sample cohort on shared infrastructure.

```bash
gatk --java-options "-Xmx12g" GenomicsDBImport \
  --genomicsdb-workspace-path cohort_db \
  --sample-name-map sample_map.txt \
  --intervals targets.bed \
  --interval-padding 1000 \
  --batch-size 50 \
  --bypass-feature-reader true \
  --genomicsdb-shared-posixfs-optimizations true \
  --overwrite-existing-genomicsdb-workspace true
```

**4.2 Joint Genotyping**

After the individual samples are consolidated into the GenomicsDB workspace, the joint genotyping step is performed. This step transitions the data from a storage format into an analysis-ready Multi-Sample VCF.

```bash
gatk --java-options "-Xmx12g" GenotypeGVCFs \
  -R GRCh38.chr.fa \
  -V gendb://cohort_db \
  -O OUTPUT_cohort_joint_calls.vcf.gz \
  --intervals targets.bed \
  --only-output-calls-starting-in-intervals true
```

The `GenotypeGVCFs` tool performs the final likelihood calculations to determine the most probable genotype for every sample at every variant site identified in the cohort.

## 5. Normalization

**5.1 Create and Activate the Environment**
```bash
# Create the environment named 'bcftools_env'
conda create -n bcftools_env -c bioconda -c conda-forge bcftools htslib -y

# Activate it
conda activate bcftools_env
```

**5.2 Run the Normalization**

Normalization is a critical step to ensure that the VCF is compatible with some other tools. These tools require parsimonious, biallelic records to accurately call star alleles and haplotypes. Because standard joint-genotyping outputs often include multiple alternate alleles on a single line (multiallelic sites) and inconsistent Indel positioning, normalization is used to "unpack" these records. 

By splitting multiallelic sites and left-aligning `Indels` against the reference, we create a standardized dataset that prevents "no-calls" and ensures every variant correctly matches known clinical definitions in the downstream Snakemake/NextFlow pipeline.

```bash
# Normalize: Left-align indels and split multiallelic sites
bcftools norm -m -any -f GRCh38.chr.fa OUTPUT_cohort_joint_calls.vcf.gz -Oz -o OUTPUT_cohort_joint_calls_norm.vcf.gz

# Index the result (required for most downstream tools)
bcftools index -t OUTPUT_cohort_joint_calls_norm.vcf.gz
```
**5.3 Normalized VCF Verification**

With your file now ready, run this one-liner to confirm the sample IDs match your expectations:

```bash
bcftools query -l OUTPUT_cohort_joint_calls_norm.vcf.gz | head -n 5
```

