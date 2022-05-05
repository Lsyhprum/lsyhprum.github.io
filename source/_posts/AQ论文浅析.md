---
title: AQ论文浅析
date: 2021-10-28 09:37:45
categories:
- [ANNS, QUANTIZATION-BASED]
tags:
- VQ, CVPR
hidden: true
---

# Introduction

加性量化（AQ）使用 **不同码本的编码和** 近似高维向量。基于这一基础思想，论文提出了一种新的向量编码和码本学习算法，能够最小化编码误差。

<!--more-->



***



# Additive quantization

高维向量的有损压缩方法可以大致分为两类。第一类是二进制编码，它将向量转换成一个短的比特序列，使用压缩向量间的汉明距离近似原始向量间的欧式距离。第二类是乘积量化，该类方法将每个向量分解为 **正交** 子空间的分量，然后对每个子空间中的低维分量使用单独的小码本进行矢量量化。结合查找表（LUT），PQ 能够非常有效地计算未压缩查询向量和大量 PQ 压缩向量间的距离和，在相同压缩率下，PQ 比二进制编码方法中的汉明距离近似要精确得多。

然而，PQ 假设划分的子空间分布相互独立，当子空间数据分布间的依赖性较强时，性能就会下降。因此该论文提出了 AQ，如下图所示，不同于 PQ，该方法没有对数据空间进行分解，码字与输入向量的长度相同（内存占用是 PQ 的 $m$ 倍）。在尽量保持计算效率的同时进一步提高了PQ的精度，可以看做是 PQ 的推广。

<img src="D:\BLOG\source\_posts\AQ.png" alt="AQ" style="zoom:70%;" />

该方法正式定义如下：对于 $D$ 维的高维向量，AQ 构建 $M$ 个码本，第 $m$ 个码本表示为$C^m$。每个码本包含 $K$ 个码字，第 $m$ 个码本中的第 $k$ 个码字表示为 $C^m_k$，不同于 PQ 中码字长度为 $D/M$，AQ 中码字长度固定为 $D$。高维向量的近似表示为 $\sum\limits_{m=1}^{M} c^m(i_m), i_m \in 1,\cdots,K$，其中 $i_m$ 表示在第 $m$ 个码本中的码字 id。

加性量化学习码本的任务可以定义为：$\min \sum^n_{j=1}||x_j-\sum_{m=1}^M c^m(i^m_j)||^2$，具体地，可以使用最小二乘法学习码本。同时为了进一步降低内存，该方法支持 PQ 和 AQ 混合使用，即首先使用 PQ 分段，然后在每段中使用 AQ。

根据实验证明，该方法在编码损失低于 PQ 及其改进算法 OPQ。

<img src="D:\BLOG\source\_posts\cmp.png" alt="cmp" style="zoom:70%;" />



***



*Bibliography*：

**Title**: [Additive Quantization for Extreme Vector Compression.](https://www.cv-foundation.org/openaccess/content_cvpr_2014/papers/Babenko_Additive_Quantization_for_2014_CVPR_paper.pdf)

**Affiliations**: *Yandex*

**Journal**: *CVPR'2014*



