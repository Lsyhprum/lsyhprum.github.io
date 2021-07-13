---
title: NSG论文浅析
date: 2021-07-05 19:49:47
tags:
- RNG
categories:
- [ANNS, GRAPH-BASED]
mathjax: true
---

# Introduction

该论文提出了一种 **新图索引结构 MRNG 及其近似版本 NSG**，该类结构具有如下四个优点：

* 确保图的连通性
* 降低平均出度
* 缩短搜索路径
* 减小索引尺寸

<!--more-->

# Monotonic Search Networks

* **Monotonic Path**

  在给定空间 $E^d$ 的有限点集 $S$ 中，$p$、$q$ 为点集合中任意两点，$G$ 为定义由 $S$ 中所有点构成的图。$ v1,v2...vk(v1=p,vk=q)$ 为从点 $p$ 到 $q$ 路径上依次经过的点，若 $∀i = 1,…,k−1,δ(v_i,q) > δ(v_{i+1},q)$ 则该路径可以视为单调路径。

* **Monotonic Search Network**

  若由 $S$ 构成的图 $G$ 中，任意两点 $p$、$q$ 之间至少存在一条单调路径，则 $G$ 为单调搜索网络。

  <img src="MSNET.png" alt="MSNET" style="zoom:67%;" />

  MSNET 具有如下几个性质：

  * 贪婪搜索可以不使用回溯找到 $G$ 中任意两点之间的单调路径
  * 期望路径长度



# Proximity Graphs

* **Delaunay Graphs**

  * 优点：单调搜索网络
  * 缺点：构建时间复杂度高，搜索效率低
  * Other:
    * KNN 图是 DG 图的近似
    * GNNS，EFANNA，IEH，DPG

* **Relative Neighborhood Graphs**

  * 优点：通过三角形选边策略降低平均出度
  * 缺点：不满足单调搜索网络
  * Other
    * FANNG，HNSW

* **Navigable Small-World Networks** 

  * 缺点：图的平均度数高；不保证图连通
  * Other
    * NSW，HNSW

* **Randomized Neighborhood Graphs**

  * 缺点：构建时间复杂度高 O($log^3n$)



# Algorithms

* **MRNG**

  为了降低图的平均出度，HNSW 和 FANNG 使用 RNG 使图变得稀疏，但 RNG 没有足够的边支撑其满足 MSNET 的性质，举个栗子，下图 a 为使用 RNG 裁边的示例，在 p 和 q 之间不存在单调路径。

  <img src="RNG vs MRNG.png" alt="RNG vs MRNG" style="zoom:80%;" />

  MRNG 试图通过一种新的边选择策略使得 RNG 满足单调搜索网络的性质。在给定空间 $E^d$ 的有限点集 $S$ 中，MRNG 中的有向边满足如下性质：任意两点 $p$、$q$ 所组成的月牙形区域 $lune_{pq} = B(p,\sigma(p,q)) \and B(q,\sigma(p,q))$ 不包含 $S$ 中的任何点，或存在一点 $r$ 位于月牙形区域 $lune_{pq}$，但 $\overrightarrow{pr} \notin MRNG$。

  MRNG 的选边策略和 RNG 区别在于，对于任何边 $pq \in MRNG，lune_{pq} \cap S$ 不一定为空。

* **NSG**

  虽然 MRNG 可以保证快速的搜索时间， 但其构建索引时间较高。因此该论文提出了 MRNG 的近似图——NSG：

  ```cpp
  Require: kNN Graph G, candidate pool size l for greedy search,
  max-out-degree m.
  Ensure: NSG with navigating node n
  calculate the centroid c of the dataset.
  r =random node.
  n =Search-on-Graph(G,r,c,l)%navigating node
  for all node v in G do
  	Search-on-Graph(G,n,v,l)
  	E =all the nodes checked along the search
  	add v’s nearest neighbors in G to E
  	sort E in the ascending order of the distance to v.
  	result set R = ∅, p0= the closest node to v in E
  	R.add(p0)
  	while !E.empty() && R.size() < m do
          p = E.front()
  		E.remove(E.front())
  		for all node r in R do
  			if edge pv conflicts with edge pr then
  				break
  			end if
  		end for
  		if no conflicts occurs then
  			R.add(p)
  		end if
  	end while
  end for
  while True do
  	build a tree with edges in NSG from root n with DFS.
  	if not all nodes linked to the tree then
  		add an edge between one of the out-of-tree nodes and
  		its closest in-tree neighbor (by algorithm 1).
  	else
  		break.
  	end if
  end while
  ```



***

*Bibliography*：

**Title**: [Fast Approximate Nearest Neighbor Search With The Navigating Spreading-out Graph.](https://doi.org/10.14778/3303753.3303754)

**Affiliations**: *Zhejiang University*

**Journal**: *VLDB'2019*

<br/>

*References*：

* [NSG : Navigating Spread-out Graph For Approximate Nearest Neighbor Search](https://github.com/ZJULearning/nsg)

