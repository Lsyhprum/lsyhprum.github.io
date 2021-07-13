---
title: NSSG论文浅析
date: 2021-07-07 17:30:54
tags:
- RNG
categories:
- [ANNS, GRAPH-BASED]
mathjax: true
---

## Introduction

该论文提出了一种 **新的图索引结构 SSG 及其近似版本 NSSG**，用于解决 NSG 存在的三个问题：

* 检索 **数据库外查询点** 没有理论保证
* **过于稀疏**
* **索引构建时间复杂度高**

<!--more-->

## Algorithms

* **SSG**

  该论文认为 NSG 和 HNSW 等算法在裁剪边时 **只考虑距离而没有考虑角度** 是有失偏颇的，基于该问题，作者提出了一种新的图索引结构——SSG。

  <img src="SSG.png" alt="SSG" style="zoom: 67%;" />

  以上图为例，对于数据集 S 中的一点 $p \in S$，计算该点与其他点的距离 $\sigma(p,\cdot)$，并升序排列。对于以 p 为起点的任意两边 $\overrightarrow{pq}$ 和 $\overrightarrow{pr}$，若 $\ang rpq < \alpha$，则视为产生冲突，并舍弃较长的边。 

  在给定空间 $E^d$ 中，SSG 中所有有向边满足 $Cone(\overrightarrow{pq},\alpha) \cap B(p,\sigma(p,q)) \cap S = \empty$ 或 $\forall r \in Cone(\overrightarrow(pq),\alpha) \cap B(p,\sigma(p,q)) \cap S, \overrightarrow{pr} \notin SSG (0 \leq \alpha \leq 60^{\circ})$。

* **NSSG**

  ```cpp
  Require: dataset D, candidate set size l, maximum out- degree r, number of navigating points s, minimum angle α.
  Ensure: an NSSG G. 
  Build an approximate kNN graph Gknn. 
  G = ∅. 
  for all node i in Gknn do 
  	P = ∅. 
  	for all neighbor n of node i do
  		P.add(n).
  		for all neighbor n' of node n do
  			P.add(n').
  		end for
  		remove the duplicated nodes in P.
  		if P.size() ≥ l then
  			break.
  		end if
  	end for
  	Perform SSG’s pruning strategy on P.
  	Update G with selected edges.
  end for 
  for all node i in G do 
  	for all node j in node i’s neighbors do
  		try adding node i to j’s neighbors according to SSG’s pruning criteria and avoid duplicates.
  		remove longer edges if exceeding max-degree r.
  	end for
  end for 
  Random select s points from the datasets as NV. 
  for all point i in NV do 
  	Strengthen the connectivity of the graph with DFS-spanning from i.
  end for 
  return G.
  ```

  



***

*Bibliography*：

**Title**: [High Dimensional Similarity Search with Satellite System Graph: Efficiency, Scalability, and Unindexed Query Compatibility.](https://ieeexplore.ieee.org/document/9383170)

**Affiliations**: *Zhejiang University*

**Journal**: *TPAMI'2021*

<br/>

*References*：

* [SSG : Satellite System Graph For Approximate Nearest Neighbor Search](https://github.com/ZJULearning/SSG)

