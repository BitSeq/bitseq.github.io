---
title: Overview of BitSeq usage
---

# Overview of BitSeq usage

BitSeq analysis consists of two stages:
* Stage 1 for expression estimation; and
* Stage 2 for differential expression estimation.

These are explained in detail below.

## Stage 1:

We want to analyse expression of human transcripts from pair-end sample *data1-1_1.fastq, data1-1_2.fastq*. First we need to download reference in form of transcript sequence and align the reads using _bowtie_. We will use hg19 ensebml transcripts:

1. Follow this link to: [UCSC Table Browser](http://genome.ucsc.edu/cgi-bin/hgTables?hgsid=214391795&clade=mammal&org=0&db=0&hgta_group=genes&hgta_track=ensGene&hgta_table=ensGene&hgta_regionType=genome&position=&hgta_outputType=sequence&hgta_outFileName=ensemblGenes.fasta) which should set you the important fields.

 * (important fields: group:"Genes and Gene Prediction Tracks", track:"Ensembl Genes", table:"ensGene", output format:"Sequence", output file:"ensemblGenes.fasta", file type returned:"gzip compressed")

1. Click "get output"
1. Select "genomic", click "submit"
1. Select "5' UTR Exons", "CDS Exons", "3' UTR Exons", "One FASTA record per gene", click "get sequence"

### Alignment

Now create index and align:
```
$ bowtie-build -f -o 2 -t 12 --ntoa ensemblGenes.fasta ensemblGenes
$ bowtie -q -v 3 --trim3 0 --trim5 0 --all -m 100 --threads 4 --sam ensemblGenes -1 data1-1_1.fastq -2 data1-1_2.fastq data1-1.sam 
# for saving alignments in BAM format use samtools
$ bowtie -q -v 3 --trim3 0 --trim5 0 --all -m 100 --threads 4 --sam ensemblGenes -1 data1-1_1.fastq -2 data1-1_2.fastq | samtools view -hSb - -o data1-1.bam
```

 * options: input Fastq, allow up to 3 mismatches, trim 0 from 3' end, trim 0 from 5' end, return all alignments, return only for reads with less than 100 alignments, use 4 cores, produce SAM formatted output
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





## Stage 2:

Using replicates from two conditions, we want to infer the transcripts with the highest probability of differential expression. Input data are MCMC samples *data1-1.rpkm*, *data1-2.rpkm* from one condition and *data2-1.rpkm*, *data2-2.rpkm* from the other condition. The programs will be using logged values of expression. 
First we compute overall mean expression (and variance) for each transcript,  which is later used to group transcripts with similar expression, the second stage works with logged expression values thus we need to use the `--log` option:
```
$ $BitSeq/getVariance --log -o data.Lmean data[12]-[12].rpkm
```

### Hyperparameters
We need to estimate expression dependent hyperparameters:
```
$ $BitSeq/estimateHyperPar --meanFile data.Lmean -o data.param data1-[12].rpkm C data2-[12].rpkm
```

 * arguments: files with samples, the letter "C" separates samples from various conditions (it's possible to use more than 2 conditions)
 * options: file with overall expression means, output file

### Condition specific expression, and PPLR

Now we can estimate condition mean expression for each transcript. And the probability of positive log ratio [PPLR](https://github.com/BitSeq/BitSeq/wiki/PPLR).
```
$ $BitSeq/estimateDE -o data -p data.param data1-[12].rpkm C data2-[12].rpkm
```

 * arguments: files with samples, the letter "C" separates samples from various conditions (it's possible to use more than 2 conditions)
 * options: output file prefix, hyperparameters previously estimated
 * adding options `--samples` tells program to output also the samples of condition mean expression in files *data-C0.est* and *data-C1.est*

Program creates file *data.pplr* which contains Probability of Positive Log Ratio, mean log2 fold change, confidence intervals, and mean condition mean expressions.
The [PPLR](https://github.com/BitSeq/BitSeq/wiki/PPLR) values can be used to rank the transcripts based on the DE belief.

## Other programs:

Short description of other useful programs provided with BitSeq, for more details run the programs with `--help` option.

<dl>
<dt>convertSamples</dt>
<dd>converts MCMC between different formats, also useful for normalisation</dd>
<dt>extractSamples</dt>
<dd>extracts expression samples of selected transcripts; useful when interested in just few transcripts</dd>
<dt>extractTranscriptInfo.py</dt>
<dd>extracts transcript information from Fasta file into `.tr` file; useful for extracting transcript-gene grouping if the Fasta file is in one of the few recognized formats</dd>
<dt>getCounts.py</dt>
<dd>converts multiple .thetaMeans files into one file containing estimated read counts for each transcript and each file</dd>
<dt>getFoldChange</dt>
<dd>compute mean fold change between two files of samples (can be either expression from `estimateExpression` or condition mean expression from `estimateDE --samples `)</dd>
<dt>getGeneExpression</dt>
<dd>converts transcript expression samples into gene expression; it needs proper transcript information file (`.tr`) with transcript-gene grouping</dd>
<dt>getWithinGeneExpression</dt>
<dd>converts transcript expression samples into relative within-gene expression; it needs proper transcript information file (`.tr`) with transcript-gene grouping</dd>
<dt>getPPLR</dt>
<dd>compute PPLR between two files of samples (can be either expression from `estimateExpression` or condition mean expression from `estimateDE --samples `)</dd>
<dt>transposeLargeFile</dt>
<dd>transposes one or more expression sample files from (N rows M columns) to (M rows N columns) and merges them into single file</dd>
</dl>
