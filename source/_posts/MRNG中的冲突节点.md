---
title: MRNG中的冲突节点
date: 2021-08-29 21:21:20
tags:
- RNG
categories:
- [ANNS, GRAPH-BASED]
mathjax: true 
---

# Introduction

该论文从理论上证明了 MRNG 在近邻图上良好的搜索性能，并该论文提出了一种称为 **冲突节点** 的 MRNG 隐藏结构，并从理论上证明了如何利用冲突节点改进基于 MRNG 的 ANNS 算法。 



<!--more-->



# MRNG

* **单调性** 

  在给定空间 $E^d$ 的有限点集 $S$ 中，$p$、$q$ 为点集合中任意两点，$G$ 为定义由 $S$ 中所有点构成的图。$ v1,v2...vk(v1=p,vk=q)$ 为从点 $p$ 到 $q$ 路径 $P$ 上依次经过的点，若 $∀i = 1,…,k−1,δ(v_i,q) > δ(v_{i+1},q)$ 则该路径 $P$ 为单调路径。**单调性保证了搜索精度**。

* **单调图**

  根据单调性的定义，如果图 $G$ 是单调的，则 $G$ 的每个子图也是单调的。值得注意的是，在基于图的 ANNS 中，搜索复杂度是访问节点之和，因此需要在保持近邻图单调性以确保搜索精度的同时，尽可能的减少边的数量。 

* **MRNG**

  对于 $S$ 中的点 $x \in S$，令 $U_x= S \backslash \{x\}$，并根据到 $x$ 的距离增序排列。对于 $y \in U_x, r \in NN(x)$，如果 $\delta(x,y) < \delta(r,y)$（即 $lune(x,y)$ 中没有 $r$） ，则将 $y$ 添加到 $x$ 的近邻中。该算法复杂度为 $O(n^2(logn+\Delta))$，其中 $\Delta$ 为 MRNG 中的最大出度。由推导可以看出，构建 MRNG 低效的原因主要来自三个方面：

  * MRNG 过高的度数 $\Delta$；
  * 图中任意两点间都需要计算距离；
  * 所有 $S\backslash \{x\}$ 中的点都需要排序。

  通过限制每个点的最大出度为 $m$，并构建候选池以限制比较点的个数 $l$，则广义 MRNG 的时间复杂度为 $O(nl(logl+m))$，其中，**$l$ 和 $m$ 的值决定了图的导航性和复杂度的权衡**。

* **广义 MRNG 的性质**

  * 理论度上界

    根据 MRNG 的定义，来自于同一个节点的任意两条边间的度数不小于 $60^{\circ}$，最大为 $O((1+6/\pi)^d)$。由此可见，MRNG 上界和数据集数量无关，但严重依赖于 $d$。

  * MRNG 度分布实验

    <img src="histogram of degree.png" alt="histogram of degree" style="zoom: 67%;" />

    根据实验显示，在相同数量的数据点上，100 维的平均度数是 10 维平均度数的 3 倍多，且100 维的最大度数是 10 维最大度数的 7.5 倍多。可以证明，在高维度构建 MRNG 比在低维度构建 MRNG 需要的时间多得多。

    <img src="ave and max degree.png" alt="ave and max degree" style="zoom: 67%;" />

    由上图可以得出以下结论：首先，随着数据量上升，平均度数不超过最大度数的三分之一，这意味着 MRNG 的平均度存在一个渐进上界，且与数据集大小无关；其次，在数据点数量固定的情况下，平均度数和最大度数都随着数据维度的增加而增加，且增加速率逐渐降低。

    综合上述实验结果，可以得出 $O((1+6/\pi)^d)$ 对于 $T(d)$ 是不紧的，这意味着在高维中构造 MRNG 可能比想象中更有效。

  * MRNG 度上限对于搜索的影响

    <img src="search acc.png" alt="search acc" style="zoom:67%;" />

    在广义 MRNG 中设置度上界不仅降低了构建和搜索时间复杂度，并且有助于提高搜索精度。此外，可以观察到一个度上限的阈值，使得搜索精度发生相变。一旦度上限大于该阈值，搜索精度不会有显著变化。论文认为，**这种相变表明了图的连通性等性质的变化**。



## Conflicting Nodes

举个栗子，令 $\overrightarrow{vu}$ 为 MRNG 中的一条边，$w$ 为由于边 $\overrightarrow{vu}$ 而无法成为 $v$ 近邻的点，即冲突节点。如图所示，虽然冲突节点 $w$ 距离查询点 $q$ 更近，但基于 MRNG 的贪婪搜索无法到达，而是陷入局部最优点 $v$。

<img src="conflicting node.png" alt="conflicting node" style="zoom:50%;" />

反过来，要检查 $w$ 的位置是否存在节点，只需检查 $v$ 在下图阴影区域中是否有邻居即可。令该区域是为 $R_w$。由于 $S$ 中比 $v$ 更接近 $q$ 的每个节点必须包含在以半径 $\delta(v,q)$ 的 $q$ 为中心的球中，因此，如果 $v$ 不是 $q$ 的全局最小值，那么 $v$ 一定在 $R_w$  内存在 $v$ 的近邻。

<img src="conflicting node2.png" alt="conflicting node2" style="zoom: 50%;" />

令 $d=\delta(v,u),\theta=\angle qvu \in [0,\pi]$。则当 $d < \delta(v,q) \cdot f(\theta)$ 时，在 $lune(v,w)$ 中存在真正近邻 $w$。其中

$$ f(\theta)=\left\{\begin{aligned}2 &, & \theta \in [0,\frac{1}{3}\pi] \\2\cos(\theta-\pi/3) & , & \theta\in(\frac{1}{3}\pi,\frac{2}{3}\pi) \\2(\cos\theta+1) & , & \theta \in (\frac{2}{3}\pi,\pi]\end{aligned}\right.$$

上述引理在二维的可视化如下，由于 $v$ 为 $q$ 的局部极小值，因此 $v$ 在以 $q$ 为圆心的圆内，没有任何邻居。若 $v$ 不是全局最小值，则必定存在 $v$ 的邻居 $u$ 被包含在阴影区域内。 

<img src="visualize.png" alt="visualize" style="zoom:67%;" />
