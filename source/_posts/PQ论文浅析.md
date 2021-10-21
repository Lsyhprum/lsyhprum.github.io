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

乘积量化（PQ）核心思想是 **将空间分解为低维子空间的笛卡尔积，并将每个子空间分别量化**。原始向量由其子空间量化编码重构成的编码表示。此外，PQ 的 **非对称计算** 通过比较 **向量和量化编码** 而非编码和编码间的近似距离提高了精度。

<!--more-->



***



# Vector quantization

最近邻搜索（NNS）是许多应用中不可获缺的一部分。然而，由于 “维度诅咒” 的原因，最近邻搜索时间复杂度极高。早期流行的 KD-tree 或其他分支定界技术在低维数据上取得了一定效果，但对于高维数据，该类方法将退化为暴力穷举，时间复杂度为 $O(nD)$。因此目前普遍通过近似最近邻搜索（ANNS）来解决这个问题，即寻找高概率的近似近邻，而非精确的近邻。

欧式距离局部敏感哈希（E2LSH）是早期流行的 ANNS 算法，并成功应用于多个领域。然而，对于真实数据，利用向量分布的启发式方法优于 LSH，典型的方法包括随机 KD-tree 和分层 K-means。这些 ANNS 算法通常基于搜索质量和效率间的权衡来进行比较，但并未考虑索引结构的内存需求，严重限制了可应用的数量规模。

基于矢量量化的 ANNS 方法通过减少表示空间的基数可以缓解内存限制的问题。其中，量化器是一个映射 $D$ 维向量 $x \in \mathbb{R}^D$ 到量化编码 $q(x) \in \mathbb{C}=\{c_i;i\in \mathbb{I}\}$的函数，其中索引 $\mathbb{I}$ 个数是有限的，**$c_i$ 被称为质心，质心的集合 $C$ 被称为码本**。映射到给定索引 $i$ 的向量集 $\mathbb{V}_i$ 被称为 Voronoi 单元，即 $\mathbb{V}_i = \{x\in\mathbb{R}^D:q(x)=c_i\}$，位于同一单元中的所有向量都由相同的质心 $c_i$ 表示。**量化器的质量通常通过输入向量 $x$ 和其量化编码 $q(x)$ 间的均方误差（MSE）测量**，即：

$$MSE(q)=\mathbb{E}_X[d(q(x),x)^2]=\int p(x)d(q(x),x)^2dx$$

然而，量化是一个破坏性的过程。想要较高的搜索质量，质心的总数 $k$ 应该足够大，这也带来了诸多问题：

* 学习量化器所需的样本数量巨大；
* 算法的复杂度高；
* 保存所有质心需要占用大量内存。

分层 K-means (HKM) 提高了构建和搜索的效率，当仍未解决上述问题；标量量化器存在较大的量化误差；**晶格量化器为均匀分布的数据提供了更好的性能，但真实数据很少满足这一条件**。在实践中，这些量化器效果均差于 K-means。



# Product Quantization

* **Product quantizers**

  乘积量化有效解决了之前矢量量化在性能和 **内存占用** 等方面的问题。对于 D 维输入向量 $x$，分为 $m$ 个不同的 $D^*$ 维子向量 $u_j(1\leq j \leq m, D^*=D/m)$，并使用 $m$ 个不同的子量化器 $q_j$ 分别量化这 $m$ 个子向量生成 $m$ 个子编码（聚类质心）。

  $\begin{matrix} \underbrace{ x_1,\cdots,x_{D^*} } \\ u_1(x) \end{matrix},\cdots,\begin{matrix} \underbrace{ x_{D-D^*+1},\cdots,x_{D} } \\ u_m(x) \end{matrix} \rightarrow q_1(u_1(x)),\cdots,q_m(u_m(x))$

  PQ 的优势在于从几个小质心集生成一组大的质心，相比于其他矢量量化算法，该算法是唯一一个可以在内存中保持较大质心个数 $k$ 的算法。PQ 中码本由对子量化器码本进行笛卡尔积生成，而量化编码则是 $m$ 个子编码的串联。假设每个子码本中质心个数为 $k^*$，则乘积量化器的码本总数为 $k=(k^*)^m$。

  <img src="pq.png" alt="pq" style="zoom:50%;" />

  在极端情况下，$m=D$，向量中每个特征都是单独量化的，则乘积量化器变成标量量化器；m=1，乘积量化器可以视为常规的 K-Means，论文中，作者尝试了多种 $k$ 和 $m$ 的组合，如下图。

  <img src="mse.png" alt="mse" style="zoom:50%;" />

* **Searching with quantization**

  该论文提出两种方法计算查询向量 $x$ 和索引向量 $y$ 间的近似欧式距离：对称距离计算（SDC）和非对称距离计算（ADC），SDC 中 $x$ 和 $y$ 都使用各自对应的量化编码进行计算，而 **ADC 使用 $x$ 的原始向量和 $y$ 对应的量化编码进行计算**。具体而言，查询向量按训练样本生成码本的过程切分成子向量，并计算各自子向量对应子空间中最近聚类中心的距离并求和相加，即可得到非对称距离。 ADC 能够在和 SDC 类似的复杂度下获得较低的距离失真，SDC 仅在内存使用上占据优势。

  <img src="adc.png" alt="pq" style="zoom:70%;" />

  作为改进，索引会保存子量化器中所有质心间的距离作为查找表，使得后续距离计算的时间复杂度降为$O(1)$。
  
* **IVFADC**

  为了避免穷举查找，该论文将倒排索引（IVF）和 ADC 相结合，共包含2层量化：

  * 第 1 层为粗量化器，通过空间划分的方法缩小搜索空间。具体而言，在原始的向量空间中，基于 K-means 聚类出 $k′$ 个簇。
  * 第 2 层为 PQ 量化器，不同于之前直接在原始向量空间中构建，此处是对原始数据 $y$ 和第一层量化生成的簇中心 $q_c(y)$ 的残差 $r(y)=y-q_c(y)$ 进行量化。相比于直接对原始数据进行量化，基于残差向量能更好地使用粗量化器所提供的信息。
  
  <img src="ivf.png" alt="ivf" style="zoom:70%;" />
  
  这种方法大大加快了搜索速度并略微提高了搜索精度，但代价是每个描述符增加几位/字节。

* **IMI**

  IVF 的通过存储每个质心附近的向量列表，可以高效地获取查询向量的近邻集。然而，该方法为了提高召回率需要增加质心，从而增加查询时间和索引构造时间。因此 [The Inverted Multi-Index](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=6915715&casa_token=3tQHSE0Mjo0AAAAA:PHcM5OUNzmiM1OIk7Wfq8-j3nydSMJnM0uMfEwuKwSbOuBim_hM6-q-ZYFHOGUfO8niG7x31a0Npp88&tag=1) 一文中提出了 IMI，能够在不增加查询时间和预处理时间的情况下产生更精细的搜索空间划分。 

  <img src="imi.png" alt="imi" style="zoom:50%;" />

  具体地，文章认为对于没有明显簇的数据，强行使用 K-Means 将导致样本划分很糟糕（簇内数据方差很大），从而在查找时丢失实际很近的数据点，如上图。因此，作者提出分别对各维度做 K-Means，查询时，根据各维度的质心检索 IMI，从而获得更准确的邻近点。

  <img src="imi search.png" alt="imi search" style="zoom:50%;" />

  举个栗子~，检索时将查询向量 $q$ 的维度均分成两个子空间并分别量化，生成子向量 $q^1, q^2$。计算子向量在对应子空间到质心的距离，$r(k)=d_1(q^1, u_{\alpha(k)}), s(k)=d_2(q^2, v_{\beta(k)})$，并根据 $r(k)+s(k)$ 进行排序。

  

***



*Bibliography*：

**Title**: [Product Quantization for Nearest Neighbor Search.](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=5432202)

**Author**: *Jeff Johnson, [Matthijs Douze](https://scholar.google.com/citations?user=0eFZtREAAAAJ&hl=en), [Hervé Jégou](https://scholar.google.com.hk/citations?user=1lcY2z4AAAAJ&hl=zh-CN)*

**Affiliations**: *INRIA*

**Journal**: *TPAMI'2010*

**Key Words**：*High-dimensional indexing, image indexing, very large databases, approximate search.*





<br/>

*References*：

[图像检索：再叙ANN Search](https://yongyuan.name/blog/ann-search.html)

[理解 product quantization 算法](http://vividfree.github.io/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/2017/08/05/understanding-product-quantization)

[The Inverted Multi-Index简析](https://zhuanlan.zhihu.com/p/130792944)

