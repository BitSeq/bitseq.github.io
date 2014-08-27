---
layout: default
title: Read alignment for BitSeq
description: Read alignment for BitSeq
---

# Overview

BitSeq requires reads aligned to a transcriptome reference.
This document explains how to create such reference from various
sources and how to align reads using variour aligners.

## Constructing transcriptome reference

The transcriptome reference should contain sequences for all RNA
species expected to be found in the sample.

* If the sample was prepared using poly-A+ selection for mRNA, you
should include sequences for protein-coding transcripts and non-coding
transcripts.
* If the sample was prepared *without* poly-A+ selection using some
rRNA depleting protocol, you can additionally include pre-mRNA
sequences to estimate expression levels for them.

### Reference from GENCODE

1. Follow either of these links to
[GENCODE human releases](http://www.gencodegenes.org/releases/) or
[GENCODE mouse releases](http://www.gencodegenes.org/mouse_releases/)
and select the desired release
1. Under "Fasta files", download "Protein-coding transcript sequences
in FASTA format" and "Long non-coding RNAs in FASTA format"

#### pre-mRNA sequences from GENCODE

In addition to the steps above

1. Under "GTF files", download "Main file, gene annotation on three levels"
1. Under "Fasta files", download "Genome sequence fasta file"
1. Run the BitSeq tool
```
gtftool -t gencode.[XXX].annotation.gtf.gz -g GRC[YYY].genome.fa.gz genes > gencode.pre_mrnas.fa
```
where you should replace `[XXX]` and `[YYY]` with the appropriate
versions for the files you downloaded.

### Reference from Ensembl

1. Go to [Ensembl website](http://www.ensembl.org/) and navigate to your favourite species
1. Under "Gene annotation", click "Download genes, cDNAs, ncRNA, proteins (FASTA)"
1. Download the file "....cdna.all.fa.gz" from directory "cdna" and gunzip it
1. Download the file "....ncrna.fa.gz" from directory "ncrna" and gunzip it
1. For human reference, it may be desirable to remove alternative haplotypes and other alternative locus entries using
```
fastagrep.sh -v 'GRCh38:[^1-9XMY]' Homo_sapiens.GRCh38.cdna.all.fa > Homo_sapiens.GRCh38.cdna.filtered.fa
fastagrep.sh -v 'GRCh38:[^1-9XMY]' Homo_sapiens.GRCh38.ncrna.all.fa > Homo_sapiens.GRCh38.ncrna.filtered.fa
```

#### pre-mRNA sequences from Ensembl

In addition to the steps above

1. Find the GTF file by replacing "fasta" in the FTP address path by "gtf", i.e. "ftp://ftp.ensembl.org/pub/release-76/fasta/homo_sapiens/" -> "ftp://ftp.ensembl.org/pub/release-76/gtf/homo_sapiens/" and download the "...gtf.gz" file
1. Returning to the Ensembl species page, under "Genome assembly", click "Download DNA sequence (FASTA)"
1. Download the file "....dna.primary_assembly.fa.gz"
1. Run the BitSeq tool
```
gtftool -t Homo_sapiens.GRCh38.76.gtf.gz -g Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz --outputFormat=ensembl genes > gencode.pre_mrnas.fa
```
where you should replace `[XXX]` and `[YYY]` with the appropriate
versions for the files you downloaded.

### Reference from UCSC genome browser

1. Follow this link to: [UCSC Table Browser](http://genome.ucsc.edu/cgi-bin/hgTables?hgsid=214391795&clade=mammal&org=0&db=0&hgta_group=genes&hgta_track=ensGene&hgta_table=ensGene&hgta_regionType=genome&position=&hgta_outputType=sequence&hgta_outFileName=ensemblGenes.fasta) which should set you the important fields.
 * (important fields: group:"Genes and Gene Prediction Tracks", track:"Ensembl Genes", table:"ensGene", output format:"Sequence", output file:"ensemblGenes.fasta", file type returned:"gzip compressed")
1. Click "get output"
1. Select "genomic", click "submit"
1. Select "5' UTR Exons", "CDS Exons", "3' UTR Exons", "One FASTA record per gene", click "get sequence"


## Read alignment

### Alignment with Bowtie 1

First create an index for your transcripts (only needs to be done once)

```
$ bowtie-build -f -o 2 -t 12 --ntoa my_transcriptome.fa,my_pre_mrnas.fa my_transcripts
```

```
$ bowtie -q -v 3 --trim3 0 --trim5 0 --all -m 100 --threads 4 --sam ensemblGenes -1 data1-1_1.fastq -2 data1-1_2.fastq data1-1.sam 
# alternatively for saving alignments in BAM format use samtools
$ bowtie -q -v 3 --trim3 0 --trim5 0 --all -m 100 --threads 4 --sam ensemblGenes -1 data1-1_1.fastq -2 data1-1_2.fastq | samtools view -hSb - -o data1-1.bam
```

 * options: input Fastq, allow up to 3 mismatches, trim 0 from 3' end, trim 0 from 5' end, return all alignments, return only for reads with less than 100 alignments, use 4 cores, produce SAM formatted output
 * samtools is a standalone software, options: include header, input is SAM, output is BAM; input is from `stdin`; output into file

### Alignment with Bowtie 2

To be written...
