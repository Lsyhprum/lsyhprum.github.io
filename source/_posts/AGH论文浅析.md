---
title: AGH论文浅析
date: 2021-12-01 19:48:03
tags:
---

# Introduction

现实世界的数据大多可以用低维流形表示，具体到 ANNS 领域中即可以通过捕捉有意义的某点的近邻表示该向量。该论文提出了一种新的基于图的哈希方法，该方法能自动发现数据中固有的领域结构来学习适当的编码（compact codes）。具体地，该论文利用锚图来获得易于处理的低秩邻接矩阵。并通过层次阈值学习过程生成二值编码。

通过无监督的方法学习适当的紧凑代码，使原始数据空间中的近邻在汉明空间中保持相邻。（谱哈希-->均匀分布）

求解拉普拉斯特征向量

样本外扩展









如何捕捉有意义的近邻。



本文主要针对 **K-近邻图（K-NNG）构建算法** 存在的几个问题：

* 需要全局索引信息，**不适合大规模数据**
* **只适用于特定度量方法**
* 构建时**较高的时间复杂度**

提出了一种基于 **局部搜索** 的 K-NNG 构建算法——**NN-Descent**。该算法可用于大规模数据、适用于任意的度量方法且构建时间复杂度降低至 O($n^{1.14}$)。

<!--more-->

# NN-Descent

* **Base Algorithm——a neighbor of a neighbor is also likely to be a neighbor**

  NN-Descent 的中心思想即通过类似梯度下降的局部搜索方法迭代地优化近邻图。

  > **Data**: dataset V, similarity oracle σ, K 
  >
  > **Result**: K-NN list B           
  >
  > begin 
  >
  > 　　B[v] ← Sample(V,K) × {∞}, ∀v ∈ V                                  // 构造随机初始图
  >
  > 　　loop 
  >
  > 　　　　R ← Reverse(B)                                                         // 更新反向集（以 v 为终点的起点集合）
  >
  > 　　　　$\overline B$[v] ← B[v] ∪ R[v], ∀v ∈ V;                                    // 获取和 v 存在连边的点集
  >
  > 　　　　c ← 0                                                                           // 初始化近邻更新次数
  >
  > 　　　　for v ∈ V do 
  >
  > 　　　　　for $u_1$ ∈ $\overline B$[v], $u_2$ ∈ $\overline B$[$u_1$] do                            // 当前点 v 与邻居的邻居进行距离比较，并更新 v 的近邻集
  >
  > 　　　　　　l ← σ(v, $u_2$) 
  >
  > 　　　　　　c ← c +UpdateNN(B[v], <$u_2$, l>)
  >
  > 　　　　return B if c = 0                                                         // 没有可更新点，终止迭代操作
  >
  > 
  >
  > function Reverse(B)                                                                 // 计算反向集
  >
  > begin 
  >
  > 　　R[v] ← {u | <v, · · ·> ∈ B[u]} ∀v ∈ V 
  >
  > return R
  >
  > 
  >
  > function UpdateNN(H, <u, l, . . .>)    							         // 返回候选集是否更新
  >
  > Update K-NN heap H; return 1 if changed, or 0 if not.

* **Local Join——let my neighbors acquaint each other**

  局部连接改进了基础算法的遍历方式。

  > // 基础算法遍历方式
  >
  > for v ∈ V do 
  >
  > ​	for $u_1$ ∈ $\overline B$[v], $u_2$ ∈ $\overline B$[$u_1$] do                               // 当前点与邻居的邻居进行比较
  >
  > ​		l ← σ(v, $u_2$) 
  >
  > ​		c ← c +UpdateNN(B[v], <$u_2$, l>)
  >
  > 
  >
  > // 局部连接遍历方式
  >
  > for v ∈ V do 
  >
  > 　　for $u_1$, $u_2$ ∈ $\overline B$[v] do                                               // 当前点的邻居间进行相互比较
  >
  > 　　　　l ← σ($u_1$, $u_2$) 
  >
  > 　　　　c ← c +UpdateNN(B[$u_1$], <$u_2$, l>)
  >
  > 　　　　c ← c +UpdateNN(B[$u_2$], <$u_1$, l>)

  举个栗子，以下图为例，基本算法通过 **当前点 $a$ ** 与 **邻居的邻居 $c$ ** 进行比较的方法更新 $a$ 的近邻集；局部连接则是对 $a$ 近邻集内 **不同的邻居 $d$ 和 $b$** 进行相似度计算和更新，类似的，a 和 c 将作为 b 的邻居相互比较。

  <img src="D:/BLOG/source/_posts/eg1.png" alt="eg1" style="zoom: 67%;" />

  通过改进，局部连接减少了迭代时访问点的个数：如果每一个点平均有 $K$ 个邻居，基础算法每次迭代将访问 $K^2$ 个点；而局部连接每次迭代只需要访问 $K$ 个点。在实际应用中有两点好处：

  * 单机部署情况下，运用 **Cache 访问的局部性原理** 提高了命中率，优化了构建效率
  * 分布式部署时，能有效减少节点间 **数据复制** 带来的性能损耗

* **Incremental Search——record neighbors' name**

  基础算法中存在大量的冗余计算，在 ANNS 中距离计算是耗费时间的主要原因，应尽量减少。

  **增量搜索** 通过在近邻数据结构中 **增加访问标记**，减少重复计算。具体而言，新近邻标记初始化为 true，在参与过至少一轮局部连接过程后，标记为 false。在局部连接过程中，两个近邻只有存在至少一个未访问才进行距离计算。

* **Sampling——old friends and new friends**

  为了进一步减少计算量，论文中通过 **随机采样** 的方法减少了需要比较的候选点个数。实现中通过取样率 ρ 来进行精度和速度的 trade-off。

* **Early Termination——done is better than perfect**

  基础算法中，若某次迭代近邻图未更新则算法终止。然而，近邻搜索算法随着召回率逼近 100%，需要更多次迭代才能获得略微提升，一般考虑到召回率和搜索性能的取舍，这些后续的提升效果很小的迭代并没有执行的必要。

  为了解决这个问题，在每次迭代中，统计近邻图更新次数 count，当 count < δKN 时终止程序。其中 δ 是精度参数，它粗略反应了由于提前终止允许错过的真正的近邻的比例。

* **Full Algorithm**

  > **Data**: dataset V, similarity oracle σ, K, ρ, δ // ρ 取样率、δ 精度参数
  >
  > **Result**: K-NN list B 
  >
  > begin 
  >
  > 　　B[v] ← Sample(V,K) × {<∞, true>} ∀v ∈ V 
  >
  > 　　loop 
  >
  > 　　　　parallel for v ∈ V do       
  >
  > 　　　　　　// 增量搜索，根据是否访问过将候选集分为两部分
  >
  > 　　　　　　old[v] ← all items in B[v] with a false flag // 增量搜索
  >
  > 　　　　　　new[v] ← ρK items in B[v] with a true flag  // 采样
  >
  > 　　　　　　Mark sampled items in B[v] as false;
  >
  > 　　　　old′ ← Reverse(old), new′ ← Reverse(new)                                            // 更新反向集
  >
  > 　　　　c ← 0 //update counter 
  >
  > 　　　　parallel for v ∈ V do                                                                                  // 混合候选集和反向集
  >
  > 　　　　　　old[v] ← old[v] ∪ Sample(old′[v], ρK)  
  >
  > 　　　　　　new[v] ← new[v] ∪ Sample(new′[v], ρK) 
  >
  > 　　　　　　for $u_1, u_2$ ∈ new[v], $u_1 < u_2$ or $u_1$ ∈ new[v], $u_2$ ∈ old[v] do   // 局部连接
  >
  > 　　　　　　　　l ← σ($u_1, u_2$) 
  >
  > 　　　　　　　　// c and B[.] are synchronized.
  >
  > 　　　　　　　　c ← c+UpdateNN(B[$u_1$], < u2, l, true >)
  >
  > 　　　　　　　　c ← c+UpdateNN(B[$u_2$], < u1, l, true >)
  >
  > 　　return B if c < δNK                                                                                              // 提前终止




# NN-Descent on High-Dimensional Data

**高维诅咒** 对各种机器学习方法和任务带来了严重挑战，其主要体现在两方面：

* **Distance concentration**

  高维数据中所有点对间的距离趋于相等；

* **Hubness**

  该性质与最近邻有关，具体表现在，维度会影响 **k-occurrences（一个点在数据集中其他点的k-近邻中出现的次数）** 的分布。随着维度的升高，将会出现 **hubs（具有非常高的 k-occurrences 的点）**。

  举个栗子~

  令 $N_k=n$ 表示为某点被算入 k 近邻的次数为 $n$，$p(N_k=n)$ 表示为 $N_k=n$ 的点占全部数据的比例。

  <img src="D:/BLOG/source/_posts/emp.png" alt="dim" style="zoom: 50%;" />

  由上图可以看出，随着维度的增加，出现明显的 **zipf 效应**。k-occurrences 很小的点数量变多；k-occurrences 很大的点数量变少，但 k-occurrences 的程度越来越大。

  ~~随着████, 富人越来越富, 但是数量变少；而穷人还是一样的穷，且数量增多。~~

NN-Descent 虽然实现了较低的构建时间复杂度，在大多数数据集上为 O($n^{1.14}$)，且易于并行。然而，该算法在本征维度较高的数据集上表现较差。

<img src="D:/BLOG/source/_posts/dim.png" alt="dim" style="zoom: 100%;" />

[NN-Descent on High-Dimensional Data](https://dl.acm.org/doi/pdf/10.1145/3227609.3227643) 认为高维数据下 K-NNG 的 **Hubness** 现象主要从以下两方面影响 NN-Descent 的性能：

* 由于 NN-Descent 每次迭代根据候选集和反向集进行优化，因此，越多次出现在其他点候选集和反向集的点更新次数越多。Hubness 较低的点出现在其他点的候选集中的概率很低，它将仅作为其他点反向集（质量较低）中的点出现并被更新。

  <img src="D:/BLOG/source/_posts/hit.png" alt="hit" style="zoom: 33%;" />

* 低 Hubness 点命中偏差较大，一些具有低 Hubness 点具有高命中，而其他低 Hubness 点可能具有非常低的命中。

  <img src="D:/BLOG/source/_posts/hit2.png" alt="hit2" style="zoom: 40%;" />

  由于 NN-Descent 初始图为随机 K-NNG，如果某点位于初始随机图中的高质量邻域中，则更可能产生高命中数。否则，该点将很快从其他点候选集中删除，并被困在不是其真实邻居的点的反向集中，导致该点更新的概率相对较低。

该论文针对 Hubness 现象对 NN-Descent 进行优化，提高了召回率，但增加了构建时间。究其本质，该算法是 **通过非均匀候选选择策略** 以实现更好的性能。



# Appendix

* **理论推导**

  * 假设数据集 $V$ 半径为  $r$，数据集 $V$ 中各点 $v$ 在 $K$ 近邻图$（K=c^3）$中的 $K$ 近邻集合由 $B_K(v)$ 表示， $v$ 邻居的邻居的集合表示为 $B'[v]=\cup_{v'\in B[v]} B[v']$；

  * 只要近邻数量 $K$ 足够多，且满足如下三个条件：

    *  $B[v]$ 包含的 $K$ 个邻居均匀分布在 $B_r(v)$ 中;
    *  每个点分布位置相互独立;
    *  $K << |B_{r/2}(v)|$;

    则**即使从随机 $K$ 近邻图开始，通过探索每个点 $v$ 邻居的邻居  $B'[v]$ ，总可以找到 $v$ 的处于半径为 $r/2$ 范围内的 $K$ 近邻 $B_{r/2}(v)$；通过不断重复上述过程，每个点的邻居集到该点的距离会不断收缩，最终，形成一个高质量近似 K-NNG**。

  * 详细证明：

    * 若 $B_{r/2}(v) $ 中的一点 $u$，$B'[v]$ 中也包含，则至少需要存在一点 $v'$，使得 $v' \in B[v]$，且 $u \in B[v']$。接下来，只需找到满足上述条件的 $v'$（$v' \in B_{r/2}(v))$，其有以下三个不等式成立：

      * $v' \in B_r(v), P\{v' \in B[v]\} \geq K/|B_r(v)|$。

        > 若 $v$ 的 $K$ 个邻居都在 $B_r(v)$ 中，则共有 $C^K_{|B_r(v)|}$ 种情况，而 $B_r(v)$ 中一点不是 $v$ 的邻居的情况共有 $C^K_{|B_r(v)|-1}$ 种，因此 $B_r(v)$ 中一点不是 $v$ 的邻居的概率为 $C^K_{|B_r(v)|-1} / C^K_{|B_r(v)|}$ ， 而 $B_r(v)$ 中一点是 $v$ 的邻居的概率为 $1-C^K_{|B_r(v)|-1} / C^K_{|B_r(v)|}$，即 $K/B_r(v)$;

      * $d(u,v') \leq d(u,v) + d(v,v') \leq r,P\{u \in B[v']\} \geq K/|B_r(v')|$。

        > 由第一条不等式可知，$B_r(v')$ 中的一点是 $v'$ 的邻居的概率为 $K/|B_r(v')|$，由于 $u$ 和 $v'$ 的距离小于等于 $r$，因此 $u$ 是 $v'$ 的邻居的概率大于等于 $K/|B_r(v')|$；

      * $|B_r(v) \leq c|B_{r/2}(v)|, |B_r(v') \leq c|B_{r/2}(v')| \leq c|B_r(v)| \leq c^2|B_{r/2}(v)|$。

        > 由于 $v'$ 在 $v$ 的 $r/2$ 超球中，$v'$ 的 $r/2$ 超球一定包含于 $v$ 的 $r$ 超球中。

      <img src="D:/BLOG/source/_posts/eg2.png" alt="eg2" style="zoom:67%;" />

    * 由以上三个不等式可得：$P\{v'\in B[v] \land u \in B[v']\} \geq K/ |B_{r/2}(v)|^2$，因此，当 $v$ 的邻居从 $B_{r/2}(v)$ 中取时，在 $B_{r/2}(v)$ 中的一点 $u$ 属于 $v$ 的邻居的邻居的概率为：$P\{u\in B'[v]\} \geq 1- (1-K/|B_{r/2}(v)|^2)^{|B_{r/2}(v)} \approx K/|B_{r/2}(v)|$

* **代码实现**

  ```cpp
  // 近邻数据结构
  struct nhood{
    std::mutex lock;				  // 锁对象
    std::vector<Neighbor> pool;	  // 结果集
    unsigned M;					  // 结果集大小，便于并发实现
    std::vector<unsigned> nn_old;	  // 已访问近邻
    std::vector<unsigned> nn_new;	  // 未访问近邻，即新的出边集合
    std::vector<unsigned> rnn_old;  // 旧反向集
    std::vector<unsigned> rnn_new;  // 新反向集，即新的采样入边集合，
    
    // 更新结果集操作，通过维护一个 大顶堆 实现
    void insert (unsigned id, float dist) {
      LockGuard guard(lock);
      if (dist > pool.front().distance) return;
      for(unsigned i=0; i<pool.size(); i++){
        if(id == pool[i].id)return;
      }
      if(pool.size() < pool.capacity()){
        pool.push_back(Neighbor(id, dist, true));
        std::push_heap(pool.begin(), pool.end());
      }else{
        std::pop_heap(pool.begin(), pool.end());
        pool[pool.size()-1] = Neighbor(id, dist, true);
        std::push_heap(pool.begin(), pool.end());
      }
    }
  
    // 局部连接
    template <typename C>
    void join (C callback) const {
      for (unsigned const i: nn_new) {
        for (unsigned const j: nn_new) {
          if (i < j) {
            callback(i, j);
          }
        }
        for (unsigned j: nn_old) {
          callback(i, j);
        }
      }
    }
  };
  
  // 集中进行局部连接，更新各点近邻，便于并发实现
  void IndexGraph::join() {
  #pragma omp parallel for default(shared) schedule(dynamic, 100)
    for (unsigned n = 0; n < nd_; n++) {
      graph_[n].join([&](unsigned i, unsigned j) {
        if(i != j){
          float dist = distance_->compare(data_ + i * dimension_, data_ + j * dimension_, dimension_);
          graph_[i].insert(j, dist);
          graph_[j].insert(i, dist);
        }
      });
    }
  }
  
  void IndexGraph::update(const Parameters &parameters) {
      
    // 清空暂存数据结构 nn_new, nn_old
    ...
      
    // 统计结果集元素数量
  #pragma omp parallel for
    for (unsigned n = 0; n < nd_; ++n) {
      auto &nn = graph_[n];
      std::sort(nn.pool.begin(), nn.pool.end());
      if(nn.pool.size()>L)nn.pool.resize(L);
      nn.pool.reserve(L);
      unsigned maxl = std::min(nn.M + S, (unsigned) nn.pool.size());
      unsigned c = 0;
      unsigned l = 0;
      while ((l < maxl) && (c < S)) {
        if (nn.pool[l].flag) ++c;
        ++l;
      }
      nn.M = l;
    }
  
  #pragma omp parallel for
    for (unsigned n = 0; n < nd_; ++n) {
      auto &nnhd = graph_[n];			  // 当前点近邻数据结构
      auto &nn_new = nnhd.nn_new;
      auto &nn_old = nnhd.nn_old;
      for (unsigned l = 0; l < nnhd.M; ++l) {		// 遍历当前点候选集
        auto &nn = nnhd.pool[l];
        auto &nhood_o = graph_[nn.id];  // 邻居的邻居
  
        if (nn.flag) {				  // 新候选点
          nn_new.push_back(nn.id);
          if (nn.distance > nhood_o.pool.back().distance) {
            // 更改候选点数据，可能产生冲突，提前加锁
            LockGuard guard(nhood_o.lock);
            // 将当前点加入候选点反向集
            if(nhood_o.rnn_new.size() < R)nhood_o.rnn_new.push_back(n);
            else{
              unsigned int pos = rand() % R;
              nhood_o.rnn_new[pos] = n;
            }
          }
          nn.flag = false;			  // 更新候选点访问状态
        } else {						  // 旧候选点
          nn_old.push_back(nn.id);
          if (nn.distance > nhood_o.pool.back().distance) {
            LockGuard guard(nhood_o.lock);
            if(nhood_o.rnn_old.size() < R)nhood_o.rnn_old.push_back(n);
            else{
              unsigned int pos = rand() % R;
              nhood_o.rnn_old[pos] = n;
            }
          }
        }
      }
      std::make_heap(nnhd.pool.begin(), nnhd.pool.end());
    }
      
    // 反向集随机采样，并混合反向集和候选集
  #pragma omp parallel for
    for (unsigned i = 0; i < nd_; ++i) {
      auto &nn_new = graph_[i].nn_new;
      auto &nn_old = graph_[i].nn_old;
      auto &rnn_new = graph_[i].rnn_new;
      auto &rnn_old = graph_[i].rnn_old;
  
      if (R && rnn_new.size() > R) {
        std::random_shuffle(rnn_new.begin(), rnn_new.end());
        rnn_new.resize(R);
      }
      nn_new.insert(nn_new.end(), rnn_new.begin(), rnn_new.end());
      if (R && rnn_old.size() > R) {
        std::random_shuffle(rnn_old.begin(), rnn_old.end());
        rnn_old.resize(R);
      }
      nn_old.insert(nn_old.end(), rnn_old.begin(), rnn_old.end());
      if(nn_old.size() > R * 2){nn_old.resize(R * 2);nn_old.reserve(R*2);}
      std::vector<unsigned>().swap(graph_[i].rnn_new);
      std::vector<unsigned>().swap(graph_[i].rnn_old);
    }
  }
  ```



***



*Bibliography*：

**Title**: [Efficient k-nearest neighbor graph construction for generic similarity measures.](https://doi.org/10.1145/1963405.1963487)

**Affiliations**: *Princeton University*

**Journal**: *WWW'2011*

**Key Words**：*arbitrary similarity measure, iterative method, k-nearest neighbor graph, information storage and retrieval.*

<br/>

*References*：

* [efanna_graph](https://github.com/ZJULearning/efanna_graph)
* [KGraph: A Library for Approximate Nearest Neighbor Search](https://github.com/aaalgo/kgraph)
* [NN-Descent构建K近邻图——论文超详细注解](https://blog.csdn.net/whenever5225/article/details/105598694)
* [[论文笔记] Hubs in Space](https://zhuanlan.zhihu.com/p/50697030)

