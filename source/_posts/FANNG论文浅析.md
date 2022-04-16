---
title: CVPR'16-Fast Approximate Nearest Neighbour Graphs
date: 2021-07-03 11:47:50
categories:
- [ANNS, GRAPH-BASED]
tags:
- RNG
- CVPR
mathjax: true
---

# Introduction

该论文提出了一种可探索图，该结构可以保证获取位于数据集一定距离阈值内查询点的真实最近邻。

<!--more-->

# Related Work

* **Hashing and quantisation techniques**

  基于 Hash 的方法通过构建多个散列表将向量映射为低维散列码，这类方法存在两个局限：

  * 高召回要求下需要大量散列表，内存开销大
  * 不适用于查询点和真实最近邻距离较远的情况

  基于量化的方法能够更好地保留数据在原始空间中的相对距离。然而，该方法和基于 Hash 的方法一样，也存在高召回率下检索效率迅速下降的问题，当 Recall 接近 1 时，候选列表所需的长度接近数据集的大小。因此，相比于基于 Hash 和量化的方法，连续划分原始数据空间的方法更有效。

* **Tree and graph techniques**

  基于树的方法提供了一种自然的方法将数据集连续划分为多个尺度的离散区域。kd-tree 根据数据的外在维度对数据进行聚类，k-means 尝试根据数据的内在结构对数据进行分组。然而，这些方法在高维数据上表现不佳，当采用错误路径时无法纠正低层的错误，只能重新遍历。

  基于图的方法能够以类似于树的方式划分搜索空间。典型的 Delaunay 图可以以确定性的方式进行探索，可以保证找到查询点的真实最近邻。然而，随着数据集维数的增加，Delaunay 图的计算效率迅速降低，因为它们很快变得几乎完全连接。K-NN 图提供了 Delaunay 图中形成的局部邻域的近似值，通过限制图中每个顶点的出度来保持有效的勘探成本，该限制牺牲了保证找到真实最近邻的能力，并且一些固有结构（例如密度的变化）不可避免地会丢失。

  对于大型数据集，计算每个顶点的 k 最近邻的计算成本很大。因此，近似 K 近邻图提供了一种替代方法来逼近 Delaunay 图中的边，为图的离线构建提供了显着的加速。然而，这类方法除了对每个顶点的度数进行限制外，图的分散式构建还进一步损害了 Delaunay  图的理想结构。这体现在需要执行回溯和多个同时进行的图探索以实现更高的平均召回率。

# Algorithms

* **Ideal graph structure**

  该论文的关键创新在于通过构建 **符合单调搜索网络的最小图** 提升搜索效率，无论哪个点作为入口点，总存在一条边通向一个更接近查询点的点，该特性确保了贪婪搜索总能找到正确的最近邻。具体地，该结构的近邻图需要保证在路由过程中和查询点的距离总是在减小，而不会被关键路径外的节点 “卡住”。举个栗子，若 $p_1$ 和 $p_2$ 间存在边，则没有必要连接一条从 $p_1$ 到比 $p_1$ 更接近 $p_2$ 的节点 $p_3$ 的边。

  <img src="occlude.png" alt="occlude" style="zoom:67%;" />

  为了提升图构建效率，应该尽量去掉冗余的“阻塞”边，考虑上图所示情形，由于 $d(p_1,p_2) < d(p1,p3)$ 且 $d(p_2,p_3) < d(p1,p3)$，根据单调搜索网络性质，总可以从 $p_1$ 通过 $p_2$ 找到 $p_3$，因此，$p_1$ 到 $p_3$ 可以看做冗余的“阻塞”边。

  > **Input**: graph vertices P                                                            // 输入 ： 向量结点 P
  >
  > **Output**: directed graph edges E                                            // 输出 ： 有向图边集 E
  >
  > for each vertex $P_i$ do
  >   
  > 　　e $\leftarrow$ empty sorted list of edges                                      // 清空边集
  > 			
  >　　for each vertex $P_j$ != $P_i$ do                                             // 计算 $P_i$ 与其他结点距离并排序
  > 			
  >　　　　add edge e($P_i$,$P_j$) sorted by distance($P_i$,$P_j$) 
  > 			
  >　　for each edge $E_j$ do
  > 			
  >　　　　u $\leftarrow$ index of end vertex of $E_j$                                // 有向边 $E_j$ 终点 u 
  > 			
  >　　　　L $\leftarrow$ length of $E_j$ 
  > 			
  >　　　　occluded ← false
  > 			
  >　　　　for each edge $E_k$ with start vertex $P_i$ do             // 遍历以 $P_i$ 为起点的边 $E_k$
  > 			
  >　　　　　　v $\leftarrow$ index of end vertex of $E_k$		         	   // 有向边 $E_k$ 终点 v
  > 			
  >　　　　　　if distance($P_u$,$P_v$) < L then                              // 若 $P_u$ 和 $P_v$ 的距离近于 $P_u$ 和 $P_i$ 的距离，则认为 $E_j$ 为阻塞边
  > 			
  >　　　　　　　　occluded $\leftarrow$ true
  > 			
  >　　　　if not occluded then
  > 			
  >　　　　　　add edge $E_j$ to E
  >
  > return E

  该论文认为理想近邻图的平均度数和本征维度相关，并使用 Hausdorff 维数进行估计：$D(r_1,r_2)=\frac{log(\frac{n(r1)}{n(r2)})}{log(\frac{r1}{r2})}$。具体地，该方法通过计算数据集中所有数据两两之间的距离，统计小于阈值 r 的成对距离的数量 $n(r)$ 估计本征维度。

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

    > **Input**: graph vertices P, directed graph edges E, query point Q, search start index v, maximum distance calculations M
    > 
    > **Output**: nearest neighbour index n
    >
    > X $\leftarrow$ empty priority queue                                   // closest to Q first 
    > 
    > add edge $e_0$ with start vertex $P_v$ to X
    > 
    > m $\leftarrow$ 1                                                                     // count distance computed to Q
    > 
    > n $\leftarrow$ v
    >
    > while m < M do
    > 
    >　　$e_i$ $\leftarrow$ remove top of X
    > 						
    >　　u $\leftarrow$ = index of end vertex of $e_i$
    > 						
    >　　if $P_u$ has not been visited yet then
    > 						
    >　　　　add edge $e_0$ with start vertex $P_u$ to X
    > 								
    >　　　　m $\leftarrow$ m+1                                               // add 1 to compute count
    > 								
    >　　　　if distance(Q,$P_u$) < distance(Q,$P_n$) then
    > 								
    >　　　　　　n $\leftarrow$ u
    > 										
    >　　v $\leftarrow$ = index of start vertex of $e_i$
    > 						
    >　　if i< number of edges with start vertex $P_v$ then
    > 						
    >　　　　add edge $e_i$+1 with start vertex $P_v$ to X
    > 				
    > return n

* **Efficient graph construction**

  基础算法时间复杂度为 O($n^2log(n)$)，该论文通过以下几种方法进行改进：

  * **构建算法**

    * 随机选择两个节点 $v_1$ 和 $v_2$，并通过贪婪搜索尝试从 $v_1$ 到 $v_2$，若未能到达 $v_2$，则 $v_2$ 与访问路径中的最后一个节点添加边。若存在遮挡问题，则需要移除较长的边以维持遮挡规则，并检查搜索路径是否仍然可达。
  
      > **Input**: graph vertices P, directed graph edges E, search start index $v_1$, search end index $v_2$
      > 
      > **Output**: directed graph edges E
      >
      > u $\leftarrow$ NaiveDownhillSearch(P, E, $P_{v_2}$ , $v_1$)
      > 
      > if u != $v_2$ then
      > 
      >　　add edge e($P_u$,$P_{v_2}$ ) to E 											// keep E sorted
      > 						
      >　　for each edge $E_i$ with start vertex $P_u$ do
      > 						
      >　　　　if e($P_u$,$P_{v_2}$) occludes $E_i$ then
      > 								
      >　　　　　　remove edge $E_i$ from E
      > 										
      >　　　　　　v $\leftarrow$ index of end vertex of $E_i$
      > 										
      >　　　　　　E $\leftarrow$ TraverseAdd(P, E, u, v)
      > 										
      >　　E $\leftarrow$ TraverseAdd(P, E, $v_2$, u) // test reverse            // test reverse
      > 		
      > return E

    * 当重复调用上述算法至效率变低时，使用当前图中的候选集基于基础算法（支持并行）构建质量更高的图。

  * **限制出度**
  
  * **入口点**
  
    选择最接近数据质心的点作为入口点，该策略通常比随机选择快几个百分点。



***

*Bibliography*：

**Title**:  [FANNG: Fast Approximate Nearest Neighbour Graphs.](https://doi.org/10.1109/CVPR.2016.616)

**Affiliations**: *Monash University*

**Journal**: *CVPR'2016*

<br/>

*References*：

* [FANNG: Fast Approximate Nearest Neighbour Graphs](https://www.youtube.com/watch?v=fSYnSUDHK4k&ab_channel=ComputerVisionFoundationVideos)

