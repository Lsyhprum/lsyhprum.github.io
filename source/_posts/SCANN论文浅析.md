---
title: SCANN论文浅析
date: 2021-11-19 10:03:15
categories:
- [ANNS, QUANTIZATION-BASED]
tags:
- VQ, ICML
mathjax: true
---

# Introduction

当前 **最大内积搜索（MIPS）量化方法** 大多以重构误差作为指标优化搜索精度，但该论文认为，重构误差并不等效于搜索精度，内积不同的点，对应的量化误差重要性不同。本文在量化时对重构误差加权，从而改善上述问题。

<img src="ScaNN tom export.gif" alt="scann" style="zoom:70%;" />

<!--more-->



---



# MIPS

以内积（IP）为距离度量的最近邻搜索被称为最大内积搜索（MIPS）。考虑具有 $n$ 个向量的数据集 $X=\{x_i\}_{i=1,2,...,n}$，每个向量 $x_i \in \mathbb{R}^d$ 位于 $d$ 维向量空间。对于给定的查询向量 $q \in \mathbb{R}^d$，MIPS 希望找到和 $q$ 具有最高内积的数据点 $x\in X$，即：

$x^*_i:=argmax_{x_i \in X} \left<q,x_i\right>$。

在数据库中向量模长相等的情况下，NNS 和 MIPS 是等价的：

 $x^*_i:=argmin_{x\in X}||q-x||^2=(||x||^2-2q^Tx)$ 



# Alogrithms

考虑将查询分布上的期望总内积量化误差最小化的量化目标：

$\mathbb{E}_q \sum_\limits{i=1}^n (\left< q,x_i\right>-\left< q,\hat{x}_i\right>)=\mathbb{E}_q \sum_\limits{i=1}^n (\left< q,x_i -\hat{x}_i\right>)=\mathbb{E}_q (x_i -\hat{x}_i)^T qq^T(x_i-\hat{x}_i)           $

在各向同性，即各个方向的方差相同的情况下，$\mathbb{E}(qq^T)=cI$，则目标函数规约为 $\mathbb{E}_q \sum_\limits{i=1}^n (\left< q,x_i -\hat{x}_i\right>)=c\sum\limits_{i=1}^n||x_i-\hat{h}_i||^2$

然而可以看出并非所有 $(x,q)$ 对都是同等重要的，具有高内积的对的近似误差要大得多，因此在实际应用中需要更加关注那些和 $x$ 有高内积查询的错误。



当x量化到~  x。结果表明，在查询分布的温和条件下，在不假设数据库点分布的情况下，最小化重构误差等价于最小化期望内积量化误差。实际上，考虑将查询分布上的期望总内积量化误差最小化的量化目标:











---

*References*：

[Announcing ScaNN: Efficient Vector Similarity Search](http://ai.googleblog.com/2020/07/announcing-scann-efficient-vector.html)

[最近邻搜索(NN)、最大内积搜索（MIPS）与(A)LSH算法](https://zhuanlan.zhihu.com/p/111502331)

[各向同性的高斯分布是指协方差矩阵是对角阵么，这个各向同性有什么直观含义呢？](https://www.zhihu.com/question/343638697/answer/1107095192)























