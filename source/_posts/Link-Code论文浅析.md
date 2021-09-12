---
title: Link&Code论文浅析
date: 2021-09-01 14:07:07
categories:
- [ANNS, GRAPH-BASED]
tags:
mathjax: true
---

# Introduction

基于图的 ANNS 算法在速度和准确率的权衡（trade-offs）中取得了卓越的效果，然而该类方法在 **内存占用和可扩展性等方面表现较差**，难以在大规模数据上应用。

该论文结合量化（内存占用小）和图（搜索效果好）方法，首先使用量化对索引向量进行压缩编码，之后 **利用图索引结构改进压缩编码**。对数据进行显著压缩的同时保持了较高精度。

<!--more-->

# Preliminary analysis

ANNS 的误差主要由两方面引起：

* **索引结构**：近似搜索方法通过缩小搜索空间，提高检索效率。搜索空间通常根据 **空间划分所产生的局部性准则选择**。如果 TP（true positive）不在搜索空间中就会发生错误。
* **压缩表示**：主流的方法包括 PCA 等传统降维方法、量化方法、以及相互结合的混合方法（PCA+量化、分层量化）。压缩表示由于距离的近似导致结果列表中的元素位置改变从而引起误差。

在此主要讨论压缩表示给 ANNS 带来的影响。

* **Centroids, neighbor or regression?**

  该论文通过 **均方误差（MSE）** 比较了几种近似向量的方法：

  * **Centroid** ：使用 K-means 算法构建一个粗码本 $\mathbb{C}$。使用码本查找最近邻 q(x) $\in \mathbb{C}$ 来近似 x。
  * **Nearest neighbor** ：使用最近邻 $n_1(x)$​ 表示。这个方法显示了在 $\mathbb{X}$ 中选择一个向量所能达到的上限。
  * **Weighted average** ：通过加权的 $x$​ 递减近邻 $N(x)=[n_1,\cdots,n_k]$​ 拟合 $x$​，即 $\overline{x}={\beta^*}^TN(x)$，其中 $\beta^*$ ​​是由所有元素 **共享的固定权重**。

  * **Regression** ：$\hat{x}=\beta(x)^TN(x)$​，其中 $\beta$​ 是 $||x-\beta(x)^TN(x)||^2$​ 的最小二乘最小值。

  <img src="CDF.png" alt="CDF" style="zoom:67%;" />

  上图显示了不同估计量的误差分布，可以得出以下三条结论：

  * **粗码本质心 $q(x)$​ 比最近邻 $n_1(x)$​ 更为有效，因此使用 HNSW 上层图作为参考点并不适合**；
  * 由 $\overline{x}$ 可知，我们可以在不增加额外存储的情况下，根据近邻信息改进对 $x$​​​ 的估计；
  * 使用回归方法可以得到更好的估计量。

* **Coding method: first approximation**

  本部分主要讨论在给定内存的情况下哪种向量压缩方法最准确。为了避免索引结构所带来的的误差影响，实验通过暴力搜索进行比较。

  实验证明，2 级编解码器比 1 级编解码器更精确，但解码的计算成本也更高。

# Algorithms

* **Vector approximation**

  首先，所有索引向量用一种 **与结构无关的编码方法压缩**（$x \in \mathbb{R}^d \rarr q(x) \in C$，其中 $C$ 是 $\mathbb{R}^d$ 的有限子集）。实验中采用二级编码，第一级使用 PQ，第二级使用 OPQ。

* **Graph-based structure**

  该论文采用了 HNSW 的索引结构，并进行部分修改使得其能够支持压缩向量。**查询或待插入的向量不进行量化，只有已经索引的元素会被量化。**

* **Refinement strategy**

  采用两阶段搜索策略。在第一阶段，仅依靠 $q(\cdot)$​​ 获取候选，索引向量由量化编码动态重建。第二阶段，根据候选集继续搜索。该论文对以上步骤提出两个优化：

  * **0-byte refinement**：不需要额外存储空间，**向量对应的压缩数据通过其在图中连接的近邻信息获得**。具体而言，对于索引向量 $x$​，若它自己和它在图中连接的 K 个近邻对应量化编码组成的矩阵为 $G(x)=[q(x),q(g_1(k)),\cdots,q(g_k(k))]$​（根据到 $x$​ 的距离进行排序，非精确近邻），则可根据该矩阵 $G(x)$​，设计一个比 $q(x)$​ 更好的压缩编码，最小化期望的平方重构损失 $L(\beta)=\sum_{x\in \mathbb{X}}||x-\beta^TG(x)||^2$​。其中，$\beta$ 由构建索引的向量学习而成，并不涉及将要使用该参数进行编码的向量。

  * **Regression codebook**：当获得每个向量 $x$​​​​ 对应的最优回归系数后，每个向量将增加 $4 \times k$​​​​ 字节的浮点值。该论文对回归权重向量进行量化，通过减少回归系数的数量降低存储开销，并提出一种 **学习码本** 的方法 $\mathbb{B}=\{\beta_1,\cdots,\beta_B\}$​​​，目标是在限制额外的内存开销的同时，尽量减少经验损失 $L'(\mathbb{B}) = \sum_{x\in \mathbb(X)}min_{\beta \in \mathbb{B}}||x-\beta^TG(x)||^2$​​​​​。

    该论文使用 K-Means 初始化回归码本，然后使用类似 EM 的算法交替执行以下两个步骤：

    * 最小化重构误差 $\beta(x)=argmin_{\beta\in\mathbb{B}}||x-\beta^TG(x)||^2$​
    * 对于每个簇，更新码本权值 $\beta_i \larr \beta^*_i$

    由于只需要 $X$ 的一个子集学习一个有代表性的码本 $\mathbb{B}$​，这个细化阶段只需要每个索引向量一个字节存储码本中选择的权重向量。（将数据分类，使用不同的小类优化不同的 beta 中一个数）

* **Product codebook**

  随着码本大小的增加，性能饱和（**理论描述**），难以继续提升。因此文中采用了类似 PQ 的方法，学习乘积回归码本。



***



*Bibliography*：

**Title**: [Link and code: Fast indexing with graphs and compact regression codes.](https://openaccess.thecvf.com/content_cvpr_2018/papers/Douze_Link_and_Code_CVPR_2018_paper.pdf)

**Affiliations**: *Facebook AI Research, Inria*

**Journal**: *CVPR ' 2018*
