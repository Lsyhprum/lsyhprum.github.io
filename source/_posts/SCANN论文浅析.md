---
title: ICML'20-Accelerating large-scale inference with anisotropic vector quantization
date: 2021-11-19 10:03:15
categories:
- [ANNS, QUANTIZATION-BASED]
tags:
- VQ
- ICML
mathjax: true
hidden: true
---

# TL;DR

当前 **最大内积搜索（MIPS）量化方法** 大多以重构误差作为指标优化搜索精度，但本文认为，重构误差并不等效于搜索精度，内积不同的点对应的量化误差重要性不同。本文在量化时对重构误差加权，从而改善上述问题。

<!--more-->



# MIPS & NNS

以内积（IP）为距离度量的近邻搜索方法被称为最大内积搜索（MIPS），其与最近邻搜索（NNS）的区别就在于距离度量不同：MIPS 以内积（IP）为距离度量，NNS 一般通过欧式距离或余弦距离进行计算。

<img src="ScaNN tom export.gif" alt="scann" style="zoom:100%;" />

一般而言，**NNS 可以简单的转换为 MIPS 问题**。具体地，对于给定空间 $\mathbb{R}^d$ 下的 $d$ 维向量 $x=\{x_1,x_2,\cdots,x_d\}$ 和查询向量 $q \in \mathbb{R}^d$，L2范数归一化后分别表示为 $x'$ 和 $q'$，其中：

$cos(x,q)=\frac{x \cdot q}{\parallel x \parallel \parallel q \parallel}=\frac{\sum_1^n{(x_i \times q_i)}}{\parallel x \parallel \parallel q \parallel}=\sum_1^n\frac{{x_i}}{\parallel x \parallel} \times \frac{{q_i}}{\parallel q \parallel}=cos(x',q')$

则，归一化向量间的余弦距离可以转化为内积：

$cos(x',q')=\frac{x' \cdot q'}{\parallel x' \parallel \parallel q' \parallel}=\frac{\sum_1^n{(x'_i \times q'_i)}}{\parallel x' \parallel \parallel q' \parallel}=\sum_1^n\frac{{x'_i}}{\parallel x' \parallel} \times \frac{{q'_i}}{\parallel q' \parallel}=\langle x',q' \rangle$

归一化向量间的 L2 和余弦距离、内积等价：

$\parallel x'-q'\parallel^2=\parallel x'\parallel^2+\parallel q'\parallel^2-2\langle x',q' \rangle=2-2\langle x',q' \rangle=2(1-cos(x',q'))$ 

**然而，从 MIPS 转化为 NNS 却存在很多问题**。现有方法试图通过对向量补全将 MIPS 转化为 NNS，即：

$\hat{x}=(x,\sqrt{1-\parallel x \parallel^2})^T, \hat{q}=(q,0)^T, \parallel x \parallel \leq 1, \parallel q \parallel = 1$

在上述条件下，l2-norm 和内积等价：

$\parallel \hat{x}-\hat{q} \parallel^2=\parallel \hat{x} \parallel ^2 + \parallel \hat{q} \parallel^2 - 2\langle \hat{x}, \hat{q} \rangle = 2-2\langle x,q \rangle$

但是上述条件较为苛刻（内积无法使用L2范数归一化，只能使用同比例缩放），因此，上述公式可以泛化为：

$\hat{x}=(x,\sqrt{1-\parallel x \parallel^2})^T, \hat{q}=(q,\sqrt{1-\parallel q \parallel^2})^T, \parallel x \parallel \leq 1, \parallel q \parallel \leq 1$

$\parallel \hat{x}-\hat{q} \parallel^2 = 2-2\langle x,q \rangle - 2 \sqrt{1-\parallel x \parallel^2} \sqrt{1-\parallel q \parallel^2} = -(\parallel x \parallel \parallel q \parallel \cos \alpha+2 \sqrt{1-\parallel x \parallel^2} \sqrt{1-\parallel q \parallel^2})$

显然，这种转换方法存在误差，并且随着维度的增加逐渐加大。具体地，维度越高，邻近（内积大）的 $x$ 和 $q$ 间的余弦值会越来越小，后项逐渐成为主导，然而，前项才是我们希望关注的。



# Alogrithms

考虑将查询分布上的期望总内积量化误差最小化的量化目标：

$\mathbb{E}_q \sum_\limits{i=1}^n (\left< q,x_i\right>-\left< q,\hat{x}_i\right>)=\mathbb{E}_q \sum_\limits{i=1}^n (\left< q,x_i -\hat{x}_i\right>)=\mathbb{E}_q (x_i -\hat{x}_i)^T qq^T(x_i-\hat{x}_i)           $

在各向同性，即各个方向的方差相同的情况下，$\mathbb{E}(qq^T)=cI$，则目标函数规约为 $\mathbb{E}_q \sum_\limits{i=1}^n (\left< q,x_i -\hat{x}_i\right>)=c\sum\limits_{i=1}^n||x_i-\hat{h}_i||^2$

然而可以看出并非所有 $(x,q)$ 对都是同等重要的，具有高内积的对的近似误差要大得多，因此在实际应用中需要更加关注那些和 $x$ 有高内积查询的错误。



当x量化到~  x。结果表明，在查询分布的温和条件下，在不假设数据库点分布的情况下，最小化重构误差等价于最小化期望内积量化误差。实际上，考虑将查询分布上的期望总内积量化误差最小化的量化目标:











---

*References*：

[Announcing ScaNN: Efficient Vector Similarity Search](http://ai.googleblog.com/2020/07/announcing-scann-efficient-vector.html)

[Non-metric Similarity Graphs for Maximum Inner Product Search](https://proceedings.neurips.cc/paper/2018/file/229754d7799160502a143a72f6789927-Paper.pdf)

[各向同性的高斯分布是指协方差矩阵是对角阵么，这个各向同性有什么直观含义呢？](https://www.zhihu.com/question/343638697/answer/1107095192)

























