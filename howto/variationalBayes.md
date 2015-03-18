---
layout: default
title: Estimate expression using Variational Bayes
description: Estimate expression using Variational Bayes
---

# Overview

In case that the user is only interested to estimate transcript expression and not to proceed to downstream analysis (e.g.: differential expression) we suggest to use the Variational Bayes (VB) version of BitSeq which is significantly faster than MCMC. The VB version of BitSeq approximates the posterior distribution of relative transcript abundance (*theta*) using a Dirichlet distribution. It has been demonstrated that this approximation is almost as accurate as MCMC for estimating the posterior means of transcript expression.

### Usage

Assume that we have already obtained a `data.prob` file of read alignments using the `parseAlignment` command. The following command will estimate the relative transcript expression using our VB algorithm.

```
$ $BitSeq/estimateVBExpression -o data data.prob
```

 * argument: the alignment probability file from pre-processing
 * options: output prefix, output type (theta, RPKM, counts, default: theta), transcript info file, number of threads to use (default: 4), optimization method (steepest, PR, FR, HS) (default: FR)

The sampler produces the file *data.m_alphas* which contains three columns. The first one corresponds to the estimated relative transcript expression levels (mean *theta*). The next two columns contain the parameters of the marginal Beta distribution describing the estimated distribution per transcript (*alpha*, *beta*). The resulting file will contains *M* lines, one for each transcript. Note that the joint distribution of all *M* transcripts is a Dirichlet with the parameters written in the second column (*alpha*).
