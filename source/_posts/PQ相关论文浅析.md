---
title: PQ相关论文浅析
date: 2021-09-02 11:11:54
categories:
- [ANNS, QUANTIZATION-BASED]
tags:
- VQ
- TPAMI
mathjax: true
---

# Introduction

乘积量化（PQ）的核心思想是 **将空间分解为低维子空间的笛卡尔积，并将每个子空间分别量化**。原始向量由对应子空间质心重构成的编码近似表示。相比于其他 ANNS 算法，该类方法 **内存开销小，数据修改灵活**，因此在工业界广泛应用。

<!--more-->



***



# Related work

最近邻搜索（NNS）是许多应用中不可获缺的一部分。然而，由于 “维度诅咒” 的原因，最近邻搜索时间复杂度极高。早期流行的 KD-tree 或其他分支定界技术在低维数据上取得了一定效果，但对于高维数据，该类方法将退化为暴力穷举。因此目前普遍通过近似最近邻搜索（ANNS）来解决这个问题，即寻找高概率而非精确的近似近邻。

欧式距离局部敏感哈希（E2LSH）是早期流行的 ANNS 算法，并成功应用于多个领域，然而对于真实数据，能够利用向量分布的启发式方法如 随机 KD-tree 和分层 K-means 在性能上优于 LSH。但是这些 ANNS 算法通常基于搜索质量和效率间的权衡来进行比较，并未考虑索引结构的内存需求，限制了实际应用中索引的数据规模。

矢量量化通过减少表示空间的基数可以缓解内存限制的问题。其中，量化器是一个映射 $D$ 维向量 $x \in \mathbb{R}^D$ 到量化编码 $q(x) \in \mathbb{C}=\{c_i;i\in \mathbb{I}\}$的函数，其中索引 $\mathbb{I}$ 个数是有限的，**$c_i$ 被称为质心，质心的集合 $C$ 被称为码本**。映射到给定索引 $i$ 的向量集 $\mathbb{V}_i$ 被称为 Voronoi 单元，即 $\mathbb{V}_i = \{x\in\mathbb{R}^D:q(x)=c_i\}$，位于同一单元中的所有向量都由相同的质心 $c_i$ 表示，即 $c_i=\int_{V_i} p(x)x dx$。**量化器的质量通常通过输入向量 $x$ 和其量化编码 $q(x)$ 间的均方误差（MSE）测量**，具体地，MSE 的计算公式如下：

$$MSE(q)=\mathbb{E}_X[d(q(x),x)^2]=\int p(x)d(q(x),x)^2dx$$

然而，量化是一个破坏性的过程。想要较高的搜索质量，质心的总数 $k$​ 必须非常大，这带来了诸多问题：

* 量化器需要数量巨大的学习样本；
* 算法的复杂度高；
* 保存所有质心需要占用大量内存。考虑一个128 维向量，量化为 64bit 的编码，因此需要构造 $k=2^{64}$ 个质心组成的码本，将占用 $2^{64} * 128 * sizeof(dtype)$ 字节内存，显然不太现实。

分层 K-means (HKM) 提高了构建和搜索的效率，当仍未解决上述问题；标量量化器存在较大的量化误差；晶格量化器为均匀分布的数据提供了更好的性能，但真实数据很少满足这一条件。在实践中，这些量化器效果均差于 K-means。



# Product Quantizers

矢量量化的主要缺陷在于：需要花费很大的代价（特别是内存开销）才能完成空间的划分，针对这个问题，PQ 利用高维的性质，能够以较小的代价完成空间划分。具体地，对于 D 维输入向量 $x$，分为 $m$ 个不同的 $D^*$ 维子向量 $u_j(1\leq j \leq m, D^*=D/m)$，并使用 $m$ 个不同的子量化器 $q_j$ 分别量化这 $m$ 个子向量生成 $m$ 个子编码（聚类质心）。PQ 中码本由对子量化器码本进行笛卡尔积生成，而量化编码则是 $m$ 个子编码的串联。假设每个子码本中质心个数为 $k^*$，则乘积量化器的码本总数为 $k=(k^*)^m$。

$\begin{matrix} \underbrace{ x_1,\cdots,x_{D^*} } \\ u_1(x) \end{matrix},\cdots,\begin{matrix} \underbrace{ x_{D-D^*+1},\cdots,x_{D} } \\ u_m(x) \end{matrix} \rightarrow q_1(u_1(x)),\cdots,q_m(u_m(x))$

乘积量化的优势在于能够从几个小质心集生成一组大的质心，相比于其他矢量量化算法，该算法是唯一一个可以在内存中保持较大质心个数 $k$ 的算法。

<img src="pq.png" alt="pq" style="zoom:70%;" />

在极端情况下，$m=D$，向量中每个特征都是单独量化的，则乘积量化器变成标量量化器；m=1，乘积量化器可以视为常规的 K-Means，论文中，作者尝试了多种 $k$ 和 $m$ 的组合，如下图。

<img src="mse.png" alt="mse" style="zoom:70%;" />

可以看出，相同编码长度下，$m$ 较少的效果更好。总体而言，$m=8,k=256$ 较为合适。

# Searching with quantization

* **ADC**

  该论文提出两种方法计算查询向量和索引中压缩向量间的近似欧式距离：对称距离计算（SDC）和非对称距离计算（ADC）。SDC 中 $x$ 和 $y$ 都使用各自对应的量化编码进行计算，而 **ADC 使用 $x$ 的原始向量和 $y$ 对应的量化编码进行计算**。具体而言，查询向量按训练样本生成码本的过程切分成子向量，并计算各自子向量对应子空间中最近聚类中心的距离并求和相加，即可得到非对称距离。 作为改进，索引会保存子量化器中所有质心间距离作为查找表，使得后续距离计算的时间复杂度降为$O(1)$。

  根据实验，ADC 能够在和 SDC 类似的复杂度下获得较低的距离失真，SDC 仅在内存使用上占据优势。

  <img src="adc.png" alt="pq" style="zoom:80%;" />
  
* **IVFADC**

  为了避免穷举查找，原论文将倒排索引（IVF）和 ADC 相结合，量化过程可以分为两层：

  * 第 1 层为粗量化器，通过空间划分的方法缩小搜索空间。具体而言，在原始的向量空间中，基于 K-means 聚类出 $k′$ 个簇。
  * 第 2 层为 PQ 量化器，不同于之前直接在原始向量空间中构建，此处是对原始数据 $y$ 和第一层量化生成的簇中心 $q_c(y)$ 的残差 $r(y)=y-q_c(y)$ 进行量化。相比于直接对原始数据进行量化，基于残差向量能更好地使用粗量化器所提供的信息。

  <img src="ivf.png" alt="ivf" style="zoom:70%;" />

  这种方法加快了搜索速度并略微提高了搜索精度，在十亿规模数据集上能够在几十毫秒内实现合理的召回率，但代价是每个描述符增加几位字节。

  <img src="IVFADC.png" alt="IVFADC" style="zoom:70%;" />

  下图反映了 IVFADC 召回率和访问近邻单元数量 $w$ 的关系，可以看出，在部分情况下仅增加 $k'$ 不增加 $w$ 将导致近邻丢失，降低了搜索效果。

  <img src="IVFADC_w.png" alt="IVFADC_w" style="zoom:70%;" />

* **IVFADC 改进算法**

  * **IMI**

    IVF 的通过存储每个质心附近的向量列表，可以高效地获取查询向量的近邻集。然而，该方法为了提高召回率需要增加质心，从而增加查询时间和索引构造时间。因此 The Inverted Multi-Index 一文中提出了 IMI，能够在不增加查询时间和预处理时间的情况下产生更精细的搜索空间划分。 

    <img src="imi.png" alt="imi" style="zoom:50%;" />

    具体地，文章认为对于没有明显簇的数据，强行使用 K-Means 将导致样本划分很糟糕（簇内数据方差很大），从而在查找时丢失实际很近的数据点，如上图。因此，作者将特征空间分解为若干正交子空间，并将每个子空间独立地划分为 Voronoi 区域。然后，每个子空间中区域的笛卡尔积形成整个特征空间的隐式划分。由于区域数量巨大，IMI 空间分区非常细粒度，每个区域只包含几个数据点。因此，IMI 在内存和运行时效率都很高的情况下，可以形成准确、简洁的候选列表。

    <img src="imi search.png" alt="imi search" style="zoom:50%;" />

    检索过程如上图所示，将查询向量 $q$ 的维度均分成两个子空间并分别量化，生成子向量 $q^1, q^2$。计算子向量在对应子空间到质心的距离，$r(k)=d_1(q^1, u_{\alpha(k)}), s(k)=d_2(q^2, v_{\beta(k)})$，并根据 $r(k)+s(k)$ 进行排序。

  * **IVFOADC+G+P**

    然而，IMI 的结构化性质也会对最终检索性能产生负面影响。Revisiting the Inverted Indices for Billion-Scale Approximate Nearest Neighbors 认为 IMI 主要有三条缺点：

    * **大多数 IMI 区域不包含点，有效区域数远小于理论区域数**。对于某些数据分布，这导致搜索过程花费大量时间访问不产生候选数据的空区域。事实上，这一缺陷的原因是 IMI 独立地学习不同子空间的 K-means 码本，而相应数据子向量的分布在实际情况中并不独立。比如，CNN 产生的描述符的不同子空间之间存在着显著的相关性。
    * 相比于 IVF，**IMI 必须访问更多的分区区域**，以积累足够的候选数量，并且转换区域的过程需要**随机内存读取**，相比于 PQ 的顺序距离计算，该操作成本更高。
    * IVF 的性能通过使用更大的码本得到显著提升，而 **IMI 在码本数量达到 $2^{14}$ 后性能达到饱和**。

    <img src="imi ivf.png" alt="imi ivf" style="zoom:40%;" />

    因此，该论文基于更大码本的 IVF 进行改进。具体地，首先根据质心构建 HNSW 索引，确定每个质心的近邻质心。之后根据近邻将 Voronoi 区域进一步细分（Grouping and pruning），缩小检索范围，实验证明该方法优于之前的 IVFADC 和 IMI 算法。

    <img src="ivfoadc.png" alt="ivfoadc" style="zoom:50%;" />


# Impact of the Component Grouping

乘积量化器将原始向量顺序拆分成子向量。然而，很多算法生成的向量中特征是有关联的，例如 GIST 描述符由 3 个 320 维的块组成，每个颜色通道一个，PQ 量化可能将这些块划分到不同的子向量，影响效果。如下图所示，将有关联的特征数据放在同一段，可以有效提升效果。

<img src="DIM-GROUPING.png" alt="DIM-GROUPING" style="zoom:70%;" />

所以在实际使用 PQ 等方法时，常会对原始数据使用 PCA 等降维方法进行预处理，一方面可以降低特征维度，另一方面能够在一定程度上降低子向量间的相关性。

<img src="itq.png" alt="itq" style="zoom:70%;" />

举个栗子~，如上面 $(a)$ 图所示，对于 PCA 降维后的二维空间，假设在做 PQ 的时候将子段数目设置为2段，即切分成 $x$ 和 $y$ 两个子向量，然后分别聚类（假设聚类中心设置为2）。通过 $(a)$ 图和 $(c)$ 图聚类的结果比较可以发现， $(a)$ 图在 $y$ 方向上聚类的效果明显差于 $(c)$ 图。这个问题可以转化为数据方差来描述：对于 PQ 量化器切分的各个子空间，我们应尽可能使得各个子空间的方差比较接近，最理想的情况是各个子空间的方差都相等，$(a)$ 图中，$x$ 和 $y$ 两个方向的方差明显有很大差异，而对于 $(c)$ 图，$x$ 和 $y$ 两个方向的方差是比较接近的。

Optimized Product Quantization for Approximate Nearest Neighbor Search 针对这一问题进行了改进。在子空间划分时，通过构造标准化正交转换矩阵使得维度组合更均衡。



***



*Bibliography*：

**Title**: [Product Quantization for Nearest Neighbor Search.](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=5432202)

**Author**: *Jeff Johnson, [Matthijs Douze](https://scholar.google.com/citations?user=0eFZtREAAAAJ&hl=en), [Hervé Jégou](https://scholar.google.com.hk/citations?user=1lcY2z4AAAAJ&hl=zh-CN)*

**Affiliations**: *INRIA*

**Journal**: *TPAMI'2011*

**Key Words**：*High-dimensional indexing, image indexing, very large databases, approximate search.*

<br/>

**Title**: [The Inverted Multi-Index.(ver.1)](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=6915715&casa_token=3tQHSE0Mjo0AAAAA:PHcM5OUNzmiM1OIk7Wfq8-j3nydSMJnM0uMfEwuKwSbOuBim_hM6-q-ZYFHOGUfO8niG7x31a0Npp88&tag=1)

**Affiliations**: *Yandex*

**Journal**: *CVPR'2012*

<br/>

**Title**: [The Inverted Multi-Index.(ver.2)](https://doi.org/10.1109/TPAMI.2014.2361319)

**Affiliations**: *Yandex*

**Journal**: *TPAMI'2015*

<br/>

**Title**: [Revisiting the Inverted Indices for Billion-Scale Approximate Nearest Neighbors](https://openaccess.thecvf.com/content_ECCV_2018/papers/Dmitry_Baranchuk_Revisiting_the_Inverted_ECCV_2018_paper.pdf)

**Affiliations**: *Yandex*

**Journal**: *ECCV'2018*

<br/>

**Title**: [Optimized Product Quantization for Approximate Nearest Neighbor Search](https://openaccess.thecvf.com/content_cvpr_2013/papers/Ge_Optimized_Product_Quantization_2013_CVPR_paper.pdf)

**Affiliations**: *University of Science and Technology of China, Microsoft Research Asia, Microsoft Research Silicon Valley*

**Journal**: *CVPR'2013*

***

*References*：

[图像检索：再叙ANN Search](https://yongyuan.name/blog/ann-search.html)

[理解 product quantization 算法](http://vividfree.github.io/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/2017/08/05/understanding-product-quantization)

[2011TPAMI-（乘积量化KNN算法）Product Quantization for Nearest Neighbor Search](https://www.jianshu.com/p/533ac746748b)

[The Inverted Multi-Index简析](https://zhuanlan.zhihu.com/p/130792944)

[Revisiting the Inverted Indices for Billion-Scale Approximate Nearest Neighbors](https://github.com/dbaranchuk/ivf-hnsw)

