---
layout: default
title: Estimate expression using Variational Bayes
description: Estimate expression using Variational Bayes
---

# Overview

In case that the user is only interested to estimate transcript expression and not to proceed to downstream analysis (e.g.: differential expression) we suggest to use the Variational Bayes (VB) version of BitSeq which is significantly faster than MCMC. The VB version of BitSeq approximates the posterior distribution of relative transcript abundance (*theta*) using a Dirichlet distribution. It has been demonstrated<sup>1, 2</sup> that this approximation is almost as accurate as MCMC for estimating the posterior means of transcript expression. 

### Basic usage

Assume that we have already obtained a `data.prob` file of read alignments using the `parseAlignment` command. The following command will estimate the relative transcript expression using our VB algorithm.

```
$ $BitSeq/estimateVBExpression -o data data.prob
```

 * argument: the alignment probability file from pre-processing
 * options: output prefix, output type (theta, RPKM, counts, default: theta), transcript info file, number of threads to use (default: 4), optimization method (steepest, PR, FR, HS) (default: FR)

The algorithm produces the file *data.m_alphas* which contains three columns. The first one corresponds to the estimated relative transcript expression levels (mean *theta*). The next two columns contain the parameters of the marginal Beta distribution describing the estimated distribution per transcript (*alpha*, *beta*). The resulting file will contains *M* lines, one for each transcript. Note that the joint distribution of all *M* transcripts is a Dirichlet with the parameters written in the second column (*alpha*).

#### RPKM output

In this case we also use the transcript information file `data.tr` produced by the `parseAlignment` command. Enable the `--samples` option in order to simulate RPKM values and then summarize the result by calling the `getVariance` command which provides the estimates of the posterior mean (and variance) of simulated RPKM values per transcript.

```
$ $BitSeq/estimateVBExpression -o data --outType RPKM -t data.tr --samples=1000 data.prob
$ $BitSeq/getVariance -o data.RPKM data.VBrpkm
```

The first column of the final output file `data.RPKM` contains the mean RPKM values (one per transcript).

### References

 1 Hensman, J., Papastamoulis, P., Glaus, P., Honkela, A., and Rattray, M. (2015). [Fast and accurate approximate inference of transcript expression from RNA-seq data.](http://bioinformatics.oxfordjournals.org/content/early/2015/09/04/bioinformatics.btv483) Bioinformatics, 10.1093/bioinformatics/btv483
 
 2 [Benchmarking experiments](https://github.com/BitSeq/BitSeqVB_benchmarking)
