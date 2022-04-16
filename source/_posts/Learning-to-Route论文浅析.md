---
title: ICML'19-Learning to Route in Similarity Graphs
date: 2022-04-07 09:12:46
tags:
- ICML
categories:
- [ANNS, GRAPH-BASED, ML]
mathjax: true
---

# Introduction

基于图的方法在检索过程中容易受到局部最优问题的影响，该论文通过结合 **近邻图全局结构信息** 提供搜索过程的最优路由决策，显著提高了整体搜索性能。

<!--more-->

# Background on TSP

TSP 是经典的路由问题。具体地，给定一系列点和边，TSP 以特定顺序遍历节点并使路径满足给定约束。例如，带有时间窗口的 TSP (TSPTW) 将时间窗口约束添加到 TSP 图中的节点，即某些节点只能在固定的时间间隔内访问；另一种变体是容量车辆路线问题 (CVRP) ，该类问题旨在为访问一组客户的车队找到最佳路线，并使每辆车都尽量满足最大承载。

近年来，以深度学习方法解决 TSP 问题成为主流。该类算法主要由 5 个步骤组成：

<img src="pipeline.png" alt="pipeline" style="zoom:100%;" />

* **Defining the problem via graphs**

  简单的 TSP 是由全连接图定义的，这显然难以适用于大型问题。因此，通过启发式算法对全连接图进行稀疏化，从而加速学习至关重要。

  <img src="step1.png" alt="step1" style="zoom:100%;" />

* **Obtaining latent embeddings for graph nodes and edges**

  将 TSP 图中的节点或边作为输入计算隐空间表示。目前常适用 GNN 获取图节点的嵌入，在每一层当中，节点从其邻居那里收集特征，再通过递归消息传递来表示局部图结构。堆叠 L 层后，网络就能从每个节点的 L 跳邻域中构建节点的特征。其中，Transformers 和 Gated Graph ConvNets 等各向异性和基于注意力的 GNN 允许每个节点根据其对解决手头任务的相对重要性来权衡其邻居节点，已成为编码路由问题的默认选择。

  <img src="step2.png" alt="step2" style="zoom:100%;" />

* **Converting embeddings into discrete solutions**

  为每个节点或每条边分配属于解集的概率，然后转换为离散决策中经典的图搜索技术，例如贪心搜索或束搜索。

  <img src="step3.png" alt="step3" style="zoom:100%;" />

* **Training the model**

  将 encoder-decoder 模型以端到端的方式进行训练，在最简单的情况下，可以通过监督学习来训练模型以产生接近最优的解。然而，为监督学习创建标记数据集是一个昂贵且耗时的过程。特别是对于大规模问题实例，最佳求解器在准确性上的保证可能不复存在，并导致用于监督训练的解决方案不精确。从实践和理论的角度来看，这远非是理想的方式。在缺乏标准解决方案的情况下，使用强化学习解决路由问题通常是一种优雅的替代方案。

# Algorithms

* **Stochastic Search Model**

  该论文将路由过程看成一个概率模型，使用可学习映射的负内积替换原始数据空间中的距离，并根据当前堆 H 中节点的 Softmax 概率分布而非距离选取下一跳节点：

  $$P(v_i|q,H, \theta)=\frac{e^{<f_{\theta}(v_i),g_{\theta}(q)>}}{\sum_{v_j \in H} e^{<f_{\theta}(v_j),g_{\theta}(q)>} }$$

  $f_{\theta}(\cdot)$ 是一个用于数据库点的神经网络，而 $g_{\theta}(\cdot)$ 是另一个用于查询的神经网络，它们的参数能够帮助随机搜索到达最近邻并将其作为 k 个输出点之一返回。其中，返回的 top-k 以内积为度量指标，因此，最后需要使用欧几里得距离进行重排。

* **Optimal routing**

  假设 $\mathcal{Ref}(H)$ 从堆 $H$ 中选取最接近实际最近邻 $v^*$ 的节点，则 $\mathcal{Ref}(H)$ 选取的节点序列构成最优路由。训练标注为预先计算的通过 BFS 返回的最小跳数。为了提高训练性能，训练过程中可能涉及的距离计算（和最小跳数差小于 5）被保存在缓存中。

* **Training objective**

  训练最佳路由的一种简单的方法（teacher forcing）是最大化对数可能性：

  $J_{naive}=E_{q,v^*}logP(Opt(q)|q,\theta)=E_{q,v^*}\sum_{v_i,H_i \in Opt(q)}logP(v_i|q,H_i,\theta)+logP(v^* \in TopK|q, V, \theta)$

  其中，$v_i,H_i \in Opt(q)$ 表示 $q$ 的最优路径上每一步的迭代顶点和堆状态，$P(v^* \in TopK|q, V, \theta)$ 是 $v*$ 被选为 top-K 之一的概率，如果 $v*$ 属于前 $k$ 个，则词查询的整体搜索成功。
  
  然而以这种方法不适用与数据库外查询，一旦该搜索算法选择非最佳节点，它会将错误的节点添加到堆 $H$中，搜索算法将进入训练期间从未出现过的状态。换句话说，上述算法没有经过训练以补偿路由错误。为了缓解该问题，该论文采用了模仿学习（imitation learning）的方式，允许选择次优节点。损失函数表示如下：
  
  $J_{naive}=E_{q,v^*}\sum_{v_i,H_i \in Search_{\theta}(q)}logP(v_i \in Ref(H_i)|q,H_i,\theta)+logP(v^* \in TopK|q, V, \theta)$
  
  <img src="compare.png" alt="compare" style="zoom:100%;" />

# Experiments

作者比较了基于原始数据点和基于学习表示（全维和降维）在固定距离计算（DCS）下的路由性能。对于全维情况，除去重排所需的计算，用于路由的距离计算为 $rDCS = (DCS-k)$；对于降维情况，$rDCS = C \times (DCS - k - d)$，其中，$C$ 为降维比例（$\times$ 2，$\times$ 4），$d$ 为查询向量在线映射花销。

通过实验，作者得出了以下几条结论：

* 如下图所示，在全维情况下，该方法优于原始方法；

  <img src="con1.png" alt="con1" style="zoom:100%;" />

* 在相同 DCS 预算内，压缩表示具有更高的召回率；

* 与 PCA 相比，学习表示具有更好的路由质量。

​	<img src="con2.png" alt="con2" style="zoom:100%;" />



***

*Bibliography*：

**Title**: [Learning to Route in Similarity Graphs.](http://proceedings.mlr.press/v97/baranchuk19a/baranchuk19a.pdf)

**Affiliations**: *Yandex*

**Journal**: *ICML'2019*

<br/>

*References*：

* [Learning to route in similarity graphs](https://github.com/dbaranchuk/learning-to-route)
* [Recent Advances in Deep Learning for Routing Problems](https://www.chaitjo.com/post/deep-learning-for-routing-problems/?continueFlag=b220d49bda26d4033730216fbc9275d5)















