# Zebrafish TU and AB Strain Genome Assembly

This repository provides the **de novo genome assemblies and gene annotations** for two zebrafish (*Danio rerio*) strains: **TU_YL** and **AB_YL**.

This page includes the following information:

- TU and AB assembly workflow
- TU and AB assembled sequences (FASTA format)
- TU and AB gene annotations (GFF3 format)

---

## Downloads

All data files are provided as release assets (files exceed GitHub's 100 MB repository limit). Please download from the [**v1.0 release page**](https://github.com/xinghui-guo/TU_YL-and-AB_YL-Assembly-genome/releases/tag/v1.0):

| File | Size | MD5 | Description |
|---|---|---|---|
| [TU_YL.fasta.gz](https://github.com/xinghui-guo/TU_YL-and-AB_YL-Assembly-genome/releases/download/v1.0/TU_YL.fasta.gz) | 387 MB | `6e576f968657737ad1346066b2c0772a` | TU_YL genome assembly |
| [TU_YL.gff.gz](https://github.com/xinghui-guo/TU_YL-and-AB_YL-Assembly-genome/releases/download/v1.0/TU_YL.gff.gz) | 23 MB | `8864703ed2b2bfef962abda8edebda98` | TU_YL gene annotation (GFF3) |
| [AB_YL.fasta.gz](https://github.com/xinghui-guo/TU_YL-and-AB_YL-Assembly-genome/releases/download/v1.0/AB_YL.fasta.gz) | 395 MB | `1daf83aec0f55cddda40f1ea9b636b2d` | AB_YL genome assembly |
| [AB_YL.gff.gz](https://github.com/xinghui-guo/TU_YL-and-AB_YL-Assembly-genome/releases/download/v1.0/AB_YL.gff.gz) | 24 MB | `94a9757a6c8592ae82c4cbf2565a91ac` | AB_YL gene annotation (GFF3) |

After downloading, verify integrity with:

```bash
md5sum -c md5sum.txt   # (checksums listed in the table above)
```

## Genome Statistics

| Assembly | Sequences | Type |
|---|---|---|
| TU_YL | 67 | chromosome-level scaffolds |
| AB_YL | 123 | chromosome-level scaffolds |

## Assembly Workflow

The TU_YL and AB_YL genomes were assembled by combining **Nanopore long reads**, **10x linked reads** and **Hi-C**, with **Bionano optical mapping** as complementary long-range support.

### Software and Versions

| Tool | Version |
|---|---|
| Guppy | v5.0.7 |
| Fastp | v0.23.4 |
| NextDenovo | v2.5.2 |
| NextPolish | v1.4.1 |
| YaHS | v1.2.2 |
| Bionano Solve | v3.8.2 |
| bwa | v0.7.17-r1188 |
| samtools | v1.17 |
| pairtools | v1.1.0 |

### NextDenovo: assembly with Nanopore long reads

Main parameters (config file):

```ini
[General]
job_type = local
job_prefix = nextDenovo
task = all
rewrite = yes
deltmp = yes
parallel_jobs = 4
input_type = raw
read_type = ont
input_fofn = input.fofn
workdir = ./nextdenovo_output

[correct_option]
read_cutoff = 1k
genome_size = 1.5g
sort_options = -m 20g -t 15
minimap2_options_raw = -t 8
pa_correction = 2
correction_options = -p 16

[assemble_option]
minimap2_options_cns = -t 8
nextgraph_options = -a 1
```

### NextPolish: polishing with short reads and long reads

Main parameters (config file):

```ini
[General]
job_type = local
job_prefix = nextPolish
task = best
rewrite = yes
deltmp = yes
rerun = 3
parallel_jobs = 10
multithread_jobs = 10
genome = TU_YL_assembly.fasta
genome_size = 1.5g
workdir = NextPolish_results
polish_options = -p {multithread_jobs}

[sgs_option]
sgs_fofn = TU_YL_sgs.fofn
sgs_options = -max_depth 100 -bwa

[lgs_option]
lgs_fofn = TU_YL_lgs.fofn
lgs_options = -min_read_len 1k -max_depth 100
lgs_minimap2_options = -x map-ont
```

### YaHS: scaffolding with Hi-C data (TU_YL shown as an example)

**1. Build the genome index with bwa**

```bash
bwa index -p prefix TU_YL.fasta
```

**2. Map Hi-C reads to the assembly**

```bash
bwa mem -t 50 -SP5M /path/to/prefix TU_HiC_pool_1.fastq TU_HiC_pool_2.fastq \
  | samtools view -@ 50 -Shb - > TU_YL.bam
```

**3. Convert BAM to pairs with pairtools**

```bash
samtools view -@ 50 -h TU_YL.bam \
  | pairtools parse -c TU_YL_genome_contig_size.txt --add-columns mapq \
  | pairtools sort --nproc 50 --memory 400G --compress-program lz4c --tmpdir tmp/ \
    --output TU_YL.sam.pairs.gz
```

**4. Remove duplicates with pairtools**

```bash
pairtools dedup --mark-dups --output-dups - --output-unmapped - \
  --output TU_YL.marked.sam.pairs.gz TU_YL.sam.pairs.gz

pairtools split --nproc-in 30 --nproc-out 30 \
  --output-pairs TU_YL.marked.sam.pairs.gz --output-sam TU_YL_unsorted.bam \
  TU_YL.marked.sam.pairs.gz

samtools sort -@ 40 -n -o TU_YL_readname_sorted.bam TU_YL_unsorted.bam
```

**5. Scaffolding with YaHS**

```bash
yahs TU_YL_genome.nextpolish.fasta -e GATC -o TU_YL_resolution \
  --file-type bam TU_YL_readname_sorted.bam \
  -r 5000,10000,20000,50000,100000,200000,500000,1000000,2000000,5000000,10000000,20000000
```

Other steps of the analysis pipelines are described in detail in the Materials and Methods section of the corresponding manuscript.

## Source of Raw Data / Reference Genomes

- **Strains**: TU (Tübingen) and AB, the two most widely used wild-type zebrafish strains, obtained from the laboratory's in-house fish facility (Y Lab, Fudan University).
- **Reference genome used for comparison**: GRCz11 (available at [Ensembl](https://ftp.ensembl.org/pub/release-113/fasta/danio_rerio/dna/)) and the mhaESC genome ([download link](https://github.com/yulab-ql/mhaESC_genome/releases/download/mT2T-Y_updte/mhaESC_v1.1_with_mT2T-Y_v1.1.251107.fasta.gz)).

## Contact

Please mail to **hongboyang@fudan.edu.cn** for any related questions.
