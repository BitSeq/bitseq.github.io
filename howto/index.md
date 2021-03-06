---
layout: default
title: Overview of BitSeq usage
description: Overview of BitSeq usage
---

# Overview of BitSeq usage

BitSeq analysis consists of two stages:

* Stage 1 for expression estimation; and
* Stage 2 for differential expression estimation (optional).

The 1st stage is explained in detail below. In case that differential expression analysis is also required see [BitSeq stage 2](http://bitseq.github.io/howto/stage2). 

## Stage 1:

We want to analyse expression of human transcripts from pair-end sample *data1-1_1.fastq, data1-1_2.fastq*. First we need to download reference in form of transcript sequence and align the reads using _bowtie_. We will use hg19 ensebml transcripts:

1. Follow this link to: [UCSC Table Browser](http://genome.ucsc.edu/cgi-bin/hgTables?hgsid=214391795&clade=mammal&org=0&db=0&hgta_group=genes&hgta_track=ensGene&hgta_table=ensGene&hgta_regionType=genome&position=&hgta_outputType=sequence&hgta_outFileName=ensemblGenes.fa) which should set you the important fields.
 * (important fields: group:"Genes and Gene Prediction Tracks", track:"Ensembl Genes", table:"ensGene", output format:"Sequence", output file:"ensemblGenes.fasta", file type returned:"gzip compressed")
1. Click "get output"
1. Select "genomic", click "submit"
1. Select "5' UTR Exons", "CDS Exons", "3' UTR Exons", "One FASTA record per gene", click "get sequence"

### Alignment

Now create index and align:

```
$ bowtie2-build -f ensemblGenes.fa ensemblGenes
$ bowtie2 -q -k 100 --threads 4 --no-mixed --no-discordant -x ensemblGenes -1 data1-1_1.fastq -2 data1-1_2.fastq -S data1-1.sam 
# for saving alignments in BAM format use samtools
$ bowtie2 -q -k 100 --threads 4 --no-mixed --no-discordant -x ensemblGenes -1 data1-1_1.fastq -2 data1-1_2.fastq | samtools view -hSb - -o data1-1.bam
```

The user can also use bowtie1 instead of bowtie2. For more details on this step see the [alignment page](http://bitseq.github.io/howto/alignment).

 * samtools is a standalone software, options: include header, input is SAM, output is BAM; input is from `stdin`; output into file



We will also need a list of transcript names with their lengths which is expected in form:
 
 *  first line contains: # M &lt;number of transcripts&gt;
 *  following lines: &lt;gene name&gt; &lt;transcript name&gt; &lt;transcript length&gt;

We will call it *ensemblGenes.tr*, this list can be created while pre-processing the alignments with option `--trInfoFile` (without gene names which are missing in the SAM header).

### Pre-processing

Now we need to pre-compute probabilities for each alignment, we expect that _BitSeq_ is compiled and located in the folder `$BitSeq`:

```
$ $BitSeq/parseAlignment  data1-1.sam -o data1-1.prob --trSeqFile ensemblGenes.fasta --trInfoFile ensemblGenes.tr --uniform --verbose
```

 * argument: the alignment file
 * options: output file, file with reference sequence, transcript information file to be created, assume uniform read distribution, verbose output
  * note: when computing non-uniform read distribution (without option `--uniform`) option `--procN` allows parallelisation. We advice not using more than default 3 CPUs as this tends to decrease the overall performance.


### Expression estimation

There are two approaches for estimating transcript expression: the first one uses MCMC sampling and the second one uses a Variational Bayes (VB) algorithm. MCMC sampling is more suitable for downstream analysis (Stage 2), while the significanlty faster VB algorithm is strongly recommended when the main aim of the analysis is expression estimation. See the next section for MCMC sampling, while for the VB algorithm see [this  link](http://bitseq.github.io/howto/variationalBayes). An extensive simulation study was conducted in order to benchmark the majority of available transcript expression estimation methods, including BitSeqVB and BitSeqMCMC. The full code is [available](https://github.com/BitSeq/BitSeqVB_benchmarking) in order to reproduce our results.



### Sampling

Now we run the sampler, which is the main part of the analysis and takes the most time.

Details: The sampler first produces `MCMC_burnIn` samples which are discarded, following by `MCMC_samplesN` samples, which are used to compute convergence statistics. These statistics are used to estimate _X_ where _X_ is the number of samples needed to produce `MCMC_samplesSave` samples which will be finally recorded. The _X_ is the major running time factor and can be decreased either by decreasing `MCMC_samplesSave` or by increasing the number of chains by `MCMC_chainsN` parameter, which can run in parallel (assuming enough CPUs).

<font color="grey">(Option `scaleReduction` invokes different convergence criterion used in previous versions of BitSeq which tends to have much longer running times. In this technique every iteration produces more samples and at the end expected scale reduction is calculated. If it is lower than `MCMC_targetScaleReduction` parameter, samples are written into files. Otherwise the number of generated samples is doubled for the next iteration.)</font>


```
$ $BitSeq/estimateExpression data1-1.prob -o data1-1 --outType RPKM -p parameters1.txt -t ensemblGenes.tr -P 2
```

 * argument: the alignment probability file from pre-processing
 * options: output prefix, output type, parameters file, transcript info file, number of threads to use (more than number of chains specified in `parameters1.txt` does not help, default number of chains is 4)
 * parameters file: contains basic parameters for the sampler, these can be also set by command line arguments `--MCMC_*`. This file is re-opened during the sampling and the parameters are updated. This allows us to change number of maximum number of samples or target scale reduction for running instance of the program.

The sampler produces two files, one is *data1-1.thetaMeans*, which contains mean thetas from every sampler and overall. The other file name depends on prefix and output type selected, in our case: *data1-1.rpkm* containing MCMC samples from all the chains.

The resulting file will contain *M* lines, one for each transcript. With each line containing `MCMC_samplesSave` expression samples. We can summarize these by computing mean (and variance).

```
$ $BitSeq/getVariance -o data1-1.mean data1-1.rpkm
```




## Other programs:

Short description of other useful programs provided with BitSeq, for more details run the programs with `--help` option.

<dl>
<dt>convertSamples</dt>
<dd>converts MCMC between different formats, also useful for normalisation</dd>
<dt>extractSamples</dt>
<dd>extracts expression samples of selected transcripts; useful when interested in just few transcripts</dd>
<dt>extractTranscriptInfo.py</dt>
<dd>extracts transcript information from Fasta file into <code>.tr</code> file; useful for extracting transcript-gene grouping if the Fasta file is in one of the few recognized formats</dd>
<dt>getCounts.py</dt>
<dd>converts multiple .thetaMeans files into one file containing estimated read counts for each transcript and each file</dd>
<dt>getFoldChange</dt>
<dd>compute mean fold change between two files of samples (can be either expression from <code>estimateExpression</code> or condition mean expression from <code>estimateDE --samples </code>)</dd>
<dt>getGeneExpression</dt>
<dd>converts transcript expression samples into gene expression; it needs proper transcript information file (<code>.tr</code>) with transcript-gene grouping</dd>
<dt>getWithinGeneExpression</dt>
<dd>converts transcript expression samples into relative within-gene expression; it needs proper transcript information file (<code>.tr</code>) with transcript-gene grouping</dd>
<dt>getPPLR</dt>
<dd>compute PPLR between two files of samples (can be either expression from <code>estimateExpression</code> or condition mean expression from <code>estimateDE --samples </code>)</dd>
<dt>transposeLargeFile</dt>
<dd>transposes one or more expression sample files from (N rows M columns) to (M rows N columns) and merges them into single file</dd>
</dl>


`getGeneExpression` example:  assume that the file `geneFile.txt` contains gene names (one per transcript - at the same order with `ensemblGenes.tr` file names). 

```
$ $BitSeq/getGeneExpression -o data1-1_Genes.rpkm -t ensemblGenes.tr -G geneFile.txt data1-1.rpkm
$ $BitSeq/getVariance -o data1-1_Genes data1-1_Genes.rpkm
```
The mean gene-level RPKM values are written to the 1st column of `data1-1_Genes`.

