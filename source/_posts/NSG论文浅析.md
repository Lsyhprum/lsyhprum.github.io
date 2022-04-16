---
title: VLDB'19-Fast Approximate Nearest Neighbor Search With The Navigating Spreading-out Graph
date: 2021-07-05 19:49:47
tags:
- RNG
- VLDB
categories:
- [ANNS, GRAPH-BASED]
mathjax: true
---

# Introduction

为了进一步提高图方法的搜索效率和可扩展性，该论文从 **连通性、平均出度、搜索路径长度** 和 **索引大小** 四方面入手，提出一种新的理论图索引结构 **MRNG** 和其近似版本 **NSG**。

<!--more-->

# PRELIMINARIES

## Monotonic Search Networks

* **Monotonic Path**

  在给定空间 $E^d$ 的有限点集 $S$ 中，$p$、$q$ 为点集合中任意两点，$G$ 为定义由 $S$ 中所有点构成的图。$ v1,v2...vk(v1=p,vk=q)$ 为从点 $p$ 到 $q$ 路径上依次经过的点，若 $∀i = 1,…,k−1,δ(v_i,q) > δ(v_{i+1},q)$ 则该路径可以视为单调路径。

* **Monotonic Search Network (MSNET)**

  若由 $S$ 构成的图 $G$ 中，任意两点 $p$、$q$ 之间至少存在一条单调路径，则 $G$ 为 MSNET。MSNET 本质上是 **保证了搜索路径长度的强连通图**，其搜索复杂度为 $O(n^{1/d}logn/\Delta r)$。

  <img src="MSNET.png" alt="MSNET" style="zoom:67%;" />

  在 **单调搜索网络中每个点** 都可以从任意起始点开始通过贪婪算法找到（数据库内查询）。

* **Delaunay Graphs (DG)**

  该结构已被证明是一个 MSNET，然而 DG 在高维数据上近乎完全连接，搜索效率显著降低。

* **Relative Neighborhood Graphs (RNG)**

  RNG 通过选边策略消除图中三角形的最长边将出度降低到 $C_d + o(1)$。然而，由于严格的边缘选择策略，RNG 没有足够的边作为 MSNET，因此无法保证路径的长度。Dearholt 等人提出 minimal MSNET，通过 向 RNG 中添加边，将其转化为 MSNET，其构建复杂度为 $O(n^2\log n+n^3)$，仍然不适用于高维和海量数据库。

* **Randomized Neighborhood Graph (RNG)**

  该方法以随机的方式构建，首先用一组凸锥划分每个节点周围的空间，然后选择每个锥中 $O(\log n)$ 个最近的节点作为其邻居。该方法搜索时间复杂度是 $O((\log n)^3)$，非常有吸引力。但是，该方法的索引复杂度很高。为了降低索引复杂度，他们提出了一种变体，称为 RNG*。 RNG\* 采用了 RNG 的边缘选择策略，并使用附加结构（KD-trees）来提高搜索性能。但是，其索引的时间复杂度仍然高达 $O(n^2)$。

# Algorithms

* **MRNG**

  为了降低图的平均出度，[HNSW](https://arxiv.org/ftp/arxiv/papers/1603/1603.09320.pdf) 和 [FANNG](https://doi.org/10.1109/CVPR.2016.616) 使用 RNG 使图变得稀疏，但 RNG 没有足够的边支撑其满足 MSNET 的性质，举个栗子，下图 a 使用 RNG 裁边，q 到 p 存在单调路径，而 p 到 q 不存在单调路径。

  <img src="RNG vs MRNG.png" alt="RNG vs MRNG" style="zoom:100%;" />

  MRNG 试图通过一种新的边选择策略使得 RNG 满足单调搜索网络的性质。在给定空间 $E^d$ 的有限点集 $S$ 中，MRNG 中的有向边满足如下性质：任意两点 $p$、$q$ 所组成的月牙形区域 $lune_{pq} = B(p,\sigma(p,q)) \cap B(q,\sigma(p,q))$ 不包含 $S$ 中的任何点，或存在一点 $r$ 位于月牙形区域 $lune_{pq}$，但 $\overrightarrow{pr} \notin MRNG$。

  MRNG 的选边策略和 RNG 区别在于，对于任何边 $pq \in MRNG，lune_{pq} \cap S$ 不一定为空。其搜索复杂度为 $O(cn^{1/d}\log n / \Delta r)$

* **NSG**

  虽然 MRNG 可以保证快速的搜索时间， 但其构建索引时间高，为 $O(n^2+n^2\log n + n^2 c)$。因此该论文提出了 MRNG 的近似图——NSG。其构建算法如下：

  * 构建近似 K-NNG；
  * 获取近似中心点。首先计算数据集质心，寻找距离数据库中距离质心最近的点作为入口导航点；
  * 将数据集中每个作为查询点，在 K-NNG 上贪婪搜索获取候选集，通过 MRNG 选边策略筛选连边；
  * 通过 DFS 保证全局连通。
  
  > **Require**: kNN Graph G, candidate pool size l for greedy search, max-out-degree m.
  > **Ensure**: NSG with navigating node n
  >
  > calculate the centroid c of the dataset.
  >
  > r =random node.
  >
  > n =Search-on-Graph(G,r,c,l)                                                                     // navigating node
  >
  > for all node v in G do
  >
  > 　　Search-on-Graph(G,n,v,l)
  >
  > 　　E =all the nodes checked along the search
  >
  > 　　add v’s nearest neighbors in G to E
  >
  > 　　sort E in the ascending order of the distance to v.
  >
  > 　　result set R = ∅, $p_0$= the closest node to v in E
  >
  > 　　R.add($p_0$)
  >
  > 　　while !E.empty() && R.size() < m do
  >
  > 　　　　p = E.front()
  >
  > 　　　　E.remove(E.front())
  >
  > 　　　　for all node r in R do
  >
  > 　　　　　　if edge $p_v$ conflicts with edge $p_r$ then
  >
  > 　　　　　　　　break
  >
  > 　　　　　　end if
  >
  > 　　　　end for
  >
  > 　　　　if no conflicts occurs then
  >
  > 　　　　　　R.add(p)
  >
  > 　　　　end if
  >
  > 　　end while
  >
  > end for
  >
  > while True do
  >
  > 　　build a tree with edges in NSG from root n with DFS.
  >
  > 　　if not all nodes linked to the tree then
  >
  > 　　　　add an edge between one of the out-of-tree nodes and its closest in-tree neighbor (by algorithm 1).
  >
  > 　　else
  >
  > 　　　　break.
  >
  > 　　end if
  >
  > end while

# Search In E-commercial Scenario

在淘宝电商场景下（128维亿级数据、每日更新），NSG 采用分布式部署（数据分块构建、32台机器、12小时）的方式，达到了 5ms 内 98% 的召回率。



***

*Bibliography*：

**Title**: [Fast Approximate Nearest Neighbor Search With The Navigating Spreading-out Graph.](https://doi.org/10.14778/3303753.3303754)

**Affiliations**: *Zhejiang University, Alibaba*

**Journal**: *VLDB'2019*

<br/>

*References*：

* [NSG : Navigating Spread-out Graph For Approximate Nearest Neighbor Search](https://github.com/ZJULearning/nsg)
* [Search Engine For AI：高维数据检索工业级解决方案](https://zhuanlan.zhihu.com/p/50143204)

