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

该论文结合量化（内存占用小）和图（搜索效果好）算法，提出了 **量化回归** 策略，**利用图结构改进压缩编码**。对数据进行显著压缩的同时提供了较高的搜索精度。

<!--more-->

# Preliminary analysis

ANNS 的误差主要由两方面引起：

* **搜索算法**

  近似搜索方法通过缩小搜索空间，提高检索效率。搜索空间通常根据 **空间划分所产生的局部性准则选择**。如果 TP（true positive）不在搜索空间中就会发生错误。

* **压缩表示**

  主流的方法包括传统降维方法（PCA等）、量化方法以及相互结合的混合方法（PCA+量化，分层量化）。压缩表示由于距离的近似导致结果列表中的元素位置改变从而引起误差。为了区分压缩表示引起的 FN（false negatives）和搜索算法所带来的的 FN，实验通过暴力搜索进行比较，结果证明 2 级编解码器比 1 级编解码器更精确，但解码的计算成本也更高。

  除此以外，该论文通过 **均方误差（MSE）** 比较了几种近似向量的方法：

  * **Centroid** ：使用 K-means 算法构建一个粗码本 $\mathbb{C}$。使用码本查找最近邻 q(x) $\in \mathbb{C}$ 来近似 x。
  * **Nearest neighbor** ：使用最近邻 $n_1(x)$​ 表示。这个方法显示了在 $\mathbb{X}$ 中选择一个向量所能达到的上限。
  * **Weighted average** ：通过加权的 $x$​ 递减近邻 $N(x)=[n_1,\cdots,n_k]$​ 拟合 $x$​，即 $\overline{x}={\beta^*}^TN(x)$，其中 $\beta^*$ ​​是由所有元素 **共享的固定权重**。

  * **Regression** ：$\hat{x}=\beta(x)^TN(x)$​，其中 $\beta$​ 是 $||x-\beta(x)^TN(x)||^2$​ 的最小二乘最小值。

  <img src="CDF.png" alt="CDF" style="zoom:80%;" />

  上图显示了不同估计量的误差分布，可以得出以下三条结论：

  * **粗码本质心 $q(x)$​ 比最近邻 $n_1(x)$​ 更为有效，因此使用 HNSW 上层图作为参考点并不适合**；
  * 由 $\overline{x}$ 可知，我们可以在不增加额外存储的情况下，根据近邻信息改进对 $x$​​​ 的估计；
  * 使用回归方法可以得到更好的估计量。

# Algorithms

* **Vector approximation**

  首先，所有索引向量用一种 **与图结构无关的编码方法压缩**（$x \in \mathbb{R}^d \rightarrow q(x) \in C$，其中 $C$ 是 $\mathbb{R}^d$ 的有限子集）。该论文采用二级编码，第一级使用 PQ，第二级使用 OPQ。

* **Graph-based structure**

  该论文采用了 HNSW 的索引结构，并进行部分修改使得其能够支持压缩向量。**查询或待插入的向量不进行量化，只有已经索引的元素会被量化。**

* **Refinement strategy**

  采用两阶段搜索策略。在第一阶段，仅依靠 $q(\cdot)$​​ 获取候选，索引向量由量化编码动态重建。第二阶段，根据候选集继续搜索。该论文对以上步骤提出两个优化：

  * **0-byte refinement**

    不需要额外存储空间，**向量对应的压缩数据通过其在图中连接的近邻信息获得**。具体而言，对于索引向量 $x$​，若它自己和它在图中连接的 K 个近邻对应量化编码组成的矩阵为 $G(x)=[q(x),q(g_1(k)),\cdots,q(g_k(k))]$​（根据到 $x$​ 的距离进行排序，非精确近邻），则可根据该矩阵 $G(x)$​，设计一个比 $q(x)$​ 更好的压缩编码，最小化期望的平方重构损失 $L(\beta)=\sum_{x\in \mathbb{X}}||x-\beta^TG(x)||^2$​，其中，$\beta$ 由构建索引的向量学习而成。

  * **Regression codebook**

    若使用每个向量 $x$ 对应的最优回归系数优化编码，将增加 $4 \times k$ 字节的浮点值。为限制额外的内存开销，该论文提出一种学习 **回归权重码本** $\mathbb{B}=\{\beta_1,\cdots,\beta_B\}$ 的方法，在减少回归权重的数量的同时，尽量减少经验损失 $L'(\mathbb{B}) = \sum_{x\in \mathbb(X)}min_{\beta \in \mathbb{B}}||x-\beta^TG(x)||^2$。

    该论文使用 K-Means 初始化回归权重码本，然后使用类似 EM 的算法优化码本。具体而言，交替执行以下两个步骤：
    
    * 最小化重构误差 $\beta(x)=argmin_{\beta\in\mathbb{B}}||x-\beta^TG(x)||^2$​
    * 对于每个聚簇，更新码本权值 $\beta_i \leftarrow \beta^*_i$
    
    由于只需要 $X$ 的一个子集学习一个有代表性的码本 $\mathbb{B}$​，这个细化阶段只需要每个索引向量一个字节存储码本中选择的权重向量。（将数据分类，使用不同的小类优化不同的 beta 中一个数）

* **Product codebook**

  随着码本大小的增加，性能饱和（**理论描述**），难以继续提升。因此文中采用了类似 PQ 的方法，学习乘积回归码本。
  
  

***



*Bibliography*：

**Title**: [Link and code: Fast indexing with graphs and compact regression codes.](https://openaccess.thecvf.com/content_cvpr_2018/papers/Douze_Link_and_Code_CVPR_2018_paper.pdf)

**Affiliations**: *Facebook AI Research, Inria*

**Journal**: *CVPR ' 2018*

