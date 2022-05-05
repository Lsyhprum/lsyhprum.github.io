---
title: VLDB'21-Hierarchical Graph Structure Based on Voronoi Diagrams for Solving Approximate Nearest Neighbor Search
date: 2022-02-18 15:14:30
categories:
- [ANNS, GRAPH-BASED]
tags:
- VQ
- VLDB
mathjax: true  
hidden: true
---

# Introduction

基于图的 ANNS 方法大多侧重于缩短搜索路径，**忽视了每一跳的计算成本**。因此，该论文提出了多级虚拟 Voronoi 图 HVS 加速 ANNS 速度。

<!--more-->

# HVS

* **PQ**

  随着层数的加深，HVS 逐渐增加乘积量化子空间的划分。在第一层，原空间 $E^d$ 被划分为两个子空间 $E^{d/2} \times E^{d/2}$，第二层划分为 $E^{d/4}\times E^{d/4}\times E^{d/4}\times E^{d/4}$，依次类推，在 $t$ 层，原空间被划分为 $2^t$ 个子空间。



<img src="hvs.png" alt="dim" style="zoom: 100%;" />

HVS 分层结构如上图所示。除底层外，每一层中的空间都被划分为虚拟 Voronoi 单元。对于顶层，这些单元构成了一个没有边的节点集，而在数据规模逐层减小的其他层中，只有高密度单元连接到下一个更深的 Voronoi 单元。同一层中，每个单元与其他相邻单元连接，也连接到下一个较低层中的单元。

在查询阶段，我们在查找离查询点最近的单元的同时，逐步降低层，并为基础层确定一组输入点。在底层，我们采用标准的搜索策略获得最终结果。

该论文认为，与现有的方法相比，HVS 距离计算效率更高，并且可以从上层返回的多个Voronoi单元中搜索基本层，以确保较高的查询精度。而NSG和HNSW由于索引的限制，都可以从单个数据点搜索基本层。

具体的，每层构建不同粒度的码本，自底向上构建的过程保证了一定的关联，数据量随着层数向下逐渐降低。





主要和 L&C 比较

多级量化

密度分配策略（路由策略）

层到层连接





* 新的层次结构 HVS，每个 HVS 由多个 Voronoi 图组成
* 密度分配策略
* HNSW 中引入边缘选择策略

























