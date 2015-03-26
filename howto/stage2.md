---
layout: default
title: BitSeq stage 2
description: Differential expression analysis
---

# BitSeq stage 2

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

