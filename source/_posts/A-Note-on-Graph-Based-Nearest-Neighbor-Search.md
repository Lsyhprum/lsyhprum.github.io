---
title: A Note on Graph-Based Nearest Neighbor Search
date: 2022-03-28 14:44:23
tags:
---

introduction

该论文提出了一种新的解释图算法表现的解释，KNN 图的聚簇系数

查询点周围的局部连接对召回有更直接的影响，高聚簇系数使得查询点最近邻中大多数位于图中的最大强连通分量中。

从算法的角度，搜索过程实际上是由两个阶段组成的，一个在 Max SCC 外，另一个在其中。

高聚类系数导致 MAX SCC 大小较大，因此在两阶段搜索过程中提供了良好的答案质量。





