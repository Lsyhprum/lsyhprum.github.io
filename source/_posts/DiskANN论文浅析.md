---
title: DiskANN论文浅析
date: 2021-07-13 16:24:09
categories:
- [ANNS, GRAPH-BASED]
tags:
- RNG
mathjax: true
---

# Introduction

多数 ANNS 算法 **索引必须存储在主存** 中，并不适用于大规模数据。因此该论文设计了一种可以利用 **固态硬盘** 的 ANNS 系统，并提出了一种改进图索引性能的方法。

<!--more-->

# Related Work

当前主流的大规模数据索引方法包括以下两种：

* 倒排索引（Inverted Index）+数据压缩
  * 典型方法：FAISS、IVFOADC+G+P
  * 优点：内存占用小
  * 缺点：数据有损压缩
* 分布式索引
  * 缺点：需要更多机器，代价较高

这些方法如果应用于 SSD 之上，性能将大幅下降。主要的限制来自于多次读取 SSD 带来的损失，因此，设计基于 SSD 驻留索引的的挑战在于如何 **减少搜索时访问 SSD 的次数**。

# Algorithms

基于减少检索时跳数的思想，该论文在裁边阶段设置了参数 $\alpha$ 用于控制长边的数量。

>Data: Database P with n points where i-th point has coords xi, parameters α, L, R
>Result: Directed graph G over P with out-degree <=R
>begin
>initialize G to a random R-regular directed graph	// 构造随机初始图
>let s denote the medoid of dataset P						 // 选取入口点
>let σ denote a random permutation of 1..n			  // 随机选取点
>for 1 ≤ i ≤ n do
>　　let [L;V] ← GreedySearch(s, $x_{σ(i)}$,1, L)					 // 贪婪搜索近邻
>　　run RobustPrune(σ(i),V, α,R) to update out-neighbors of σ(i)	 // 裁边
>　　for all points j in $N_{out}(σ(i))$ do								// 添加反向边，若出度大于 R 继续裁边
>　　　　if |$N_{out}(j)$ ∪ {σ(i)}| > R then
>　　　　　　run RobustPrune(j, $N_{out}(j)$ ∪ {σ(i)}, α,R) to update out-neighbors of j
>　　　　else
>　　　　　　update $N_{out}(j)$ ← $N_{out}(j)$ ∪ σ(i)

VAMANA 整体流程需要执行两边上述算法：第一次 $\alpha$ 定义为 1；第二次通过设置 $\alpha \geq 1$ 从而增加长边。

和 NSG 相比，VAMANA 主要增加了长边，并且 VAMANA 初始图为随机图，该论文认为相比于近似 K 近邻图，随机图构建速度更快，且能构建出质量更好的图。
