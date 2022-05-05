---
title: ICCV'19-Unsupervised Neural Quantization for Compressed-Domain Similarity Search
date: 2022-04-10 15:58:11
categories:
- [ANNS, QUANTIZATION-BASED]
tags:
- ICCV
- VQ
mathjax: true
hidden: true
---

# Introduction

该论文提出了一种基于 **多码本量化** 的无监督 DNN 架构。

<!--more-->

# Related work

现有的降维方法可以分为两个独立的研究方向：

* **二进制散列**：将原始向量映射到汉明空间，并尽量降低在原始空间中相近的数据在汉明空间中的距离，该类方法受益于现代 CPU 的架构，可以进行高效的计算；
* **多码本量化**：通过对来自几个不相交码本的码字进行求和或串联近似原始向量，这类方法通常不涉及查询端的信息丢失，优于二进制散列方法，具体地，可以分为两种实现：
  * **乘积量化（PQ）**：使用多个低维空间的笛卡尔积近似原始空间，其性能依赖于划分空间的独立性；
  * **非正交量化**：通过码字求和而非乘积近似原始向量，常用的方法包括残差量化（RQ）、加性量化（AQ）和复合量化（CQ），该类方法具有更高的精度。

# Algorithm

* **Model**

  现有的量化方法包括两个基本模块：编码器和距离函数。编码器 $f(x):\mathcal{R}^D \rightarrow \{1,\dots,K\}^M$ 将向量 $x$ 映射为编码 $i$；距离函数 $d(q,i): \mathcal{R}^D \times \{1,\dots,K\}^M \rightarrow \mathcal{R}$ 评估查询向量和编码数据库的距离。具体地，模型架构如下图所示：

  <img src="arch.png" alt="MSNET" style="zoom:67%;" />

  其中，$net(x)=[net(x)_1,\dots,net(x)_M]$ 为向量到 $M$ 个学习子空间乘积的映射，$c_{mk}$ 为来自第 $m$ 个码本的第 $k$ 个码字，被 $c_{mk}$ 编码的概率为：

  $$p(c_{mk}|x)=\frac{exp(\left \langle net(x)_m, c_{mk} \right \rangle / \tau_{m})}{\sum_j exp(\left \langle net(x)_m, c_{mj} \right \rangle / \tau_{m})}$$

  $c_{mk}$ 和 $\tau_{m}$ 为正则模型参数，并使用反向传播对其进行优化，假设不同码本间的码字条件独立，则损失函数为：

  $f(x)=[argmax_{c_{1k}}\left \langle net(x)_1, c_{1k} \right \rangle, \dots, argmax_{c_{Mk}}\left \langle net(x)_M, c_{Mk} \right \rangle]$

* **Training**

  论文中采用了两种距离函数训练模型，第一距离函数在原始空间中定义，使用类似自动编码器的目标进行训练：

  然而这种方法不能保证最小化这个目标会导致好的候选会进入重排阶段，因此，论文中采用最小化学习空间中的三元组损失函数，使得其更接近真实近邻而非负样本









