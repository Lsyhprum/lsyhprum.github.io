---
title: FANNG论文浅析
date: 2021-07-03 11:47:50
categories:
- [ANNS, GRAPH-BASED]
tags:
- RNG
mathjax: true
---

# Introduction

该论文提出了一种 **有向图结构**，该结构具有两个优点：

* 最小化与树结构相关的回溯成本
* 更好的利用本征维度信息，提升了算法效率

<!--more-->

# Algorithms

* **Ideal graph structure**

  该论文的关键创新在于通过构建 **符合单调搜索网络的最小图** 提升搜索效率。首先，单调搜索网络无论哪个点作为入口点，总存在一条边通向一个更接近查询点的点，该特性确保了贪婪搜索总能找到正确的最近邻。

  <img src="occlude.png" alt="occlude" style="zoom:67%;" />

  其次，为了提升图构建效率，应该尽量去掉冗余的“阻塞”边。举个栗子，考虑上图所示情形，由于 $d(p_1,p_2) < d(p1,p3)$ 且 $d(p_2,p_3) < d(p1,p3)$，根据单调搜索网络性质，总可以从 $p_1$ 通过 $p_2$ 找到 $p_3$，因此，$p_1$ 到 $p_3$ 可以看做冗余的“阻塞”边。

  ```cpp
  Input: graph vertices P // 输入 ： 向量结点 P
  Output: directed graph edges E  // 输出 ： 有向图边集 E
  
  for each vertex Pi do
    e ← empty sorted list of edges  // 清空边集
    for each vertex Pj != Pi do 
      add edge e(Pi,Pj) sorted by distance(Pi,Pj) // 计算 Pi 与其他结点距离并排序
    for each edge ej do
      u ← index of end vertex of ej // 有向边 ej 终点 u 
      L ← length of ej 
      occluded ← false
      for each edge Ek with start vertex Pi do  // 遍历以 Pi 为起点的边 ek
        v ← index of end vertex of Ek			  // 有向边 ek 终点 v
        if distance(Pu,Pv) < L then // 若 Pu 和 Pv 的距离近与 Pu 和 Pi 的距离，则认为 ej 为阻塞边
          occluded ← true
      if not occluded then
        add edge ej to E
  return E
  ```

* **Intrinsic dimensionality and vertex degree**

  该论文认为上述理想图结构所组成的近邻图的平均度数和本征维度相关。该论文本征维度使用 Hausdorff 维数进行计算：$D(r_1,r_2)=\frac{log(\frac{n(r1)}{n(r2)})}{log(\frac{r1}{r2})}$。该方法通过计算数据集中所有点两两之间的距离，并统计小于阈值 r 的成对距离的数量 $n(r)$ 估计本征维度。

  <img src="hausdorff.png" alt="hausdorff" style="zoom:67%;" />

* **Making nearest neighbor guarantees**

  上述算法只能保证通过贪婪算法找到数据库内查询点（查询点和近邻图中一个结点相同）的最近邻，为了 **确保数据库外查询点也能保持较好的查询效果**，该论文提出了两个改进方法：

  * **改进图构建算法**

    原先算法单纯考虑点和点间的关系，因此只能保证找到数据库内查询点的最近邻。改进方法中**当查询点位于某点 $\tau$ 范围内，视该点为查询点的最近邻**。

    在这种情况下，当 $d(p_1,p_2) < d(p_1,p_3)$ 且 $d(p_2,p_3)^2 < d(p_1,p_3)^2-2\tau d(p_1,p_2)$，视为 $p_1$ 到 $p_2$ 的边阻塞了 $p_1$ 到 $p_3$ 的边。

    如下图所示，尽管 $p_4$ 更接近 $p_2$，但在改进算法下，$p_1$ 到 $p_4$ 的边仍存在。

    <img src="occlude2.png" alt="occlude2" style="zoom:67%;" />

    值得注意的是，这种改进算法显著地增加了平均度数，导致搜索效率的降低。

  * **回溯搜索算法**

    为了确保数据库外查询点的查询效果。改进搜索算法通过维护一个优先队列记录距离查询点最近的点，当继续查询无法找到距离查询点更近的近邻时，通过回溯继续探索其他点。

    ```cpp
    Input: graph vertices P, directed graph edges E, query point Q, search start index v, maximum distance calculations M
    Output: nearest neighbour index n
    
    X ← empty priority queue // closest to Q first 
    add edge e0 with start vertex Pv to X
      m ← 1  // count distance computed to Q
      n ← v
    
    while m < M do
      ei ← remove top of X
      u ← = index of end vertex of ei
      if Pu has not been visited yet then
        add edge e0 with start vertex Pu to X
        m ← m+1 // add 1 to compute count
        if distance(Q,Pu) < distance(Q,Pn) then
          n ← u
      v ← = index of start vertex of ei
      if i< number of edges with start vertex Pv then
        add edge ei+1 with start vertex Pv to X
    return n
    ```

* **Efficient graph construction**

  基础算法时间复杂度为 O($n^2log(n)$)，该论文通过以下几种方法进行改进：

  * 重复执行如下算法构建初始图：

    ```cpp
    Input: graph vertices P, directed graph edges E, search start index v1, search end index v2
    Output: directed graph edges E
    
    u ← NaiveDownhillSearch(P, E, Pv2 , v1)
    if u != v2 then
      add edge e(Pu,Pv2 ) to E // keep E sorted
      for each edge Ei with start vertex Pu do
        if e(Pu,Pv2) occludes Ei then
          remove edge Ei from E
          v ← index of end vertex of Ei
          E ← TraverseAdd(P, E, u, v)
      E ← TraverseAdd(P, E, v2, u) // test reverse
    return E
    ```

  * 基于初始图中的候选集，使用基础算法构建质量更高的图

  * 选择最接近数据质心的点作为入口点





***

*Bibliography*：

**Title**:  [FANNG: Fast Approximate Nearest Neighbour Graphs.](https://doi.org/10.1109/CVPR.2016.616)

**Affiliations**: *Monash University*

**Journal**: *CVPR'2016*