---
title: PQ论文浅析
date: 2021-09-02 11:11:54
categories:
- [ANNS, QUANTIZATION-BASED]
tags:
- VECTOR QUANTIZATION
mathjax: true
---

# Introduction

乘积量化（PQ）核心思想是 **将空间分解为低维子空间的笛卡尔积，并将每个子空间分别量化**。向量由其对应的多个子空间量化编码组成的编码表示，可以有效地估计出两个矢量之间的欧氏距离。此外，PQ 的 **非对称计算** 通过比较 **向量和量化编码** 而非编码和编码间的近似距离提高了精度。实验结果表明，PQ 能够有效地搜索最近邻。

<!--more-->



***



# Vector quantization

最近邻（NN）搜索是许多应用中不可获取的一部分。然而，由于 “维度诅咒” 的原因，最近邻搜索时间复杂度极高。早期流行的 KD-tree 或其他分支定界技术在低维数据上取得了一定效果，但对于高维数据，该类方法并不比暴力穷举更有效，其时间复杂度为 $O(nD)$。因此目前普遍通过近似最近邻搜索（ANNS）来解决这个问题，即寻找高概率的近似近邻，而非精确的近邻。

欧式距离局部敏感哈希（E2LSH）是早期流行的 ANNS 算法，并成功应用于多个领域。然而，对于真实数据，利用向量分布的启发式方法优于 LSH，典型的方法包括随机 KD-tree 和分层 K-means。这些 ANNS 算法通常是基于搜索质量和效率间的权衡来进行比较，并未考虑索引结构的内存需求，严重限制了可应用的数量规模。

基于量化的 ANNS 方法通过减少表示空间的基数可以缓解内存限制的问题。量化器是一个映射 $D$ 维向量 $x \in \mathbb{R}^D$ 到量化编码 $q(x) \in \mathbb{C}=\{c_i;i\in \mathbb{I}\}$的函数，其中索引 $\mathbb{I}$ 个数是有限的，**$c_i$ 被称为质心，质心的集合 $C$ 被称为码本**。映射到给定索引 $i$ 的向量集 $\mathbb{V}_i$ 被称为 Voronoi 单元，即 $\mathbb{V}_i = \{x\in\mathbb{R}^D:q(x)=c_i\}$。量化器的 k 个单元构成 $\mathbb{R}^D$ 的一个分区，位于同一单元中的所有向量都由相同的质心 $c_i$ 表示。**量化器的质量通常通过输入向量 $x$ 和其量化编码 $q(x)$ 间的均方误差（MSE）测量**，即：

$$MSE(q)=\mathbb{E}_X[d(q(x),x)^2]=\int p(x)d(q(x),x)^2dx$$

然而，量化是一个破坏性的过程，为了提高搜索质量，质心的总数 k 应该足够大，这也带来了诸多问题：

* 学习量化器所需的样本数量巨大；
* 算法的复杂度高；
* 保存所有质心需要占用大量内存。

分层 K-means (HKM) 提高了构建和搜索的效率，当仍未解决上述问题；标量量化器存在较大的量化误差；**晶格量化器为均匀分布的数据提供了更好的性能，但真实数据很少满足这一条件**。在实践中，这些量化器效果均差于 K-means。



# Algorithms

* **Product quantizers**

  乘积量化有效解决了之前矢量量化在性能和 **内存占用** 等方面的问题。对于 D 维输入向量 $x$，分为 $m$ 个不同的 $D^*$ 维子向量 $u_j(1\leq j \leq m, D^*=D/m)$，并使用 $m$ 个不同的子量化器 $q_j$ 分别量化这 $m$ 个子向量生成 $m$ 个子编码（聚类质心）。

  $\begin{matrix} \underbrace{ x_1,\cdots,x_{D^*} } \\ u_1(x) \end{matrix},\cdots,\begin{matrix} \underbrace{ x_{D-D^*+1},\cdots,x_{D} } \\ u_m(x) \end{matrix} \rarr q_1(u_1(x)),\cdots,q_m(u_m(x))$

  PQ 的优势在于从几个小质心集生成一组大的质心，相比于其他矢量量化算法，该算法是唯一一个可以在内存中保持较大质心个数 $k$ 的算法。PQ 中码本由对子量化器码本进行笛卡尔积生成，而量化编码则是 $m$ 个子编码的串联。假设每个子码本中质心个数为 $k^*$，则乘积量化器的码本总数为 $k=(k^*)^m$。

  <img src="pq.png" alt="pq" style="zoom:50%;" />

  在极端情况下，$m=D$，向量中每个特征都是单独量化的，则乘积量化器变成标量量化器；m=1，乘积量化器可以视为 常规的 K-Means。

* **Searching with quantization**

  该论文提出两种方法计算查询向量 $x$ 和索引向量 $y$ 间的近似欧式距离：对称距离计算（SDC）和非对称距离计算（ADC），SDC 中 $x$ 和 $y$ 都使用各自对应的量化编码进行计算，而 **ADC 使用 $x$ 的原始向量和 $y$ 对应的量化编码进行计算**。具体而言，查询向量按训练样本生成码本的过程切分成子向量，并计算各自子向量对应子空间中最近聚类中心的距离并求和相加，即可得到非对称距离。 

  <img src="adc.png" alt="pq" style="zoom:50%;" />

  ADC 能够在和 SDC 类似的复杂度下获得较低的距离失真，SDC 仅在内存使用上占据优势。



***



*Bibliography*：

**Title**: [Product Quantization for Nearest Neighbor Search.](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=5432202)

**Affiliations**: *INRIA*

**Journal**: *TPAMI'2010*

**Key Words**：*High-dimensional indexing, image indexing, very large databases, approximate search.*

<br/>

*References*：

[图像检索：再叙ANN Search](https://yongyuan.name/blog/ann-search.html)

