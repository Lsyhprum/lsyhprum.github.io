---
title: NN-Descent论文浅析
date: 2021-07-01 19:05:36
categories:
- [ANNS, GRAPH-BASED]
tags:
- K-NNG
mathjax: true
---

### Introduction

该论文提出了一个 **基于局部搜索的K-近邻图（K-NNG）构建算法**，用于解决之前 K-NNG 构建算法存在的两个问题：

* 需要全局索引信息，算法无法并行执行，因此**不适合大规模数据**
* **只适用于特定度量方法**



<!--more-->

**Title**: [Efficient k-nearest neighbor graph construction for generic similarity measures.](https://doi.org/10.1145/1963405.1963487)

**Affiliations**: *Princeton University*

**Journal**: *WWW ' 2011*

**Key Words**：*arbitrary similarity measure, iterative method, k-nearest neighbor graph, information storage and retrieval.*



### Algorithms

* **Base Algorithm——a neighbor of a neighbor is also likely to be a neighbor**

  NN-Descent 的中心思想即通过类似梯度下降的局部搜索逐步优化近邻图。

  > **Data**: dataset V, similarity oracle σ, K 
  >
  > **Result**: K-NN list B           
  >
  > begin 
  >
  > ​	B[v] ← Sample(V,K) × {∞}, ∀v ∈ V                                  // 构造随机初始图
  >
  > ​	loop 
  >
  > ​		R ← Reverse(B)                                                             // 更新反向（入边）集
  >
  > ​		$\overline B$[v] ← B[v] ∪ R[v], ∀v ∈ V;                                       // 获取和 v 相连的点（无论出边入边）
  >
  > ​		c ← 0                                                                              // 初始化近邻更新次数标志 
  >
  > ​		for v ∈ V do 
  >
  > ​			for $u_1$ ∈ $\overline B$[v], $u_2$ ∈ $\overline B$[$u_1$] do                               // 当前点与邻居的邻居进行比较
  >
  > ​				l ← σ(v, $u_2$) 
  >
  > ​				c ← c +UpdateNN(B[v], <$u_2$, l>)
  >
  > ​		return B if c = 0                                                             // 未进行更新操作，返回结果集
  >
  > 
  >
  > function Reverse(B)                                                              // 计算反向近邻集
  >
  > begin 
  >
  > ​	R[v] ← {u | <v, · · ·> ∈ B[u]} ∀v ∈ V 
  >
  > return R
  >
  > 
  >
  > function UpdateNN(H, <u, l, . . .>)    // return whether candidates have updated
  >
  > Update K-NN heap H; return 1 if changed, or 0 if not.

* **Local Join——let my neighbors acquaint each other**

  基本算法通过 **当前点** 与 **所有邻居的邻居** 进行比较的方法更新当前点的结果集。

  > for v ∈ V do 
  >
  > ​	for $u_1$ ∈ $\overline B$[v], $u_2$ ∈ $\overline B$[$u_1$] do                               // 当前点与邻居的邻居进行比较
  >
  > ​		l ← σ(v, $u_2$) 
  >
  > ​		c ← c +UpdateNN(B[v], <$u_2$, l>)

  **局部连接** 对近邻遍历访问的方法进行了改进，对当前点邻居集内每一对不同的邻居进行相似度计算和更新。

  > for v ∈ V do 
  >
  > ​	for $u_1$, $u_2$ ∈ $\overline B$[v] do                                               // 当前点的邻居互相进行比较
  >
  > ​		l ← σ($u_1$, $u_2$) 
  >
  > ​		c ← c +UpdateNN(B[$u_1$], <$u_2$, l>)
  >
  > ​		c ← c +UpdateNN(B[$u_2$], <$u_1$, l>)

  举个栗子，如果每一个对象的邻居的个数平均为 K，基础算法迭代为  **a -> b -> c**，一次迭代每个对象将访问 $K^2$ 个点；而局部连接变为了 **b <- a -> c**，每次迭代只需要访问 K 个点。这样改进的好处主要有两点：

  * 单机部署情况下，运用 **Cache 访问的局部性原理** 提升了 Cache 命中率 
  * 分布式部署时，能减少机器之间 **数据复制**

* **Incremental Search——record neighbors' name**

  基础算法中存在大量的冗余计算，在 ANNS 中距离计算是耗费时间的主要原因，应尽量减少。

  **增量搜索** 通过在近邻数据结构中增加访问标记，减少重复计算。新近邻标记初始化为 true，在参与过至少一轮局部连接过程后，标记为 false。在局部连接过程中，两个近邻只有存在至少一个未访问才进行距离计算。

* **Sampling——old friends and new friends**

  为了进一步减少计算量，论文中通过 **随机采样** 的方法减少了需要比较的候选点个数。实现中通过取样率 ρ 来进行精度和速度的 trade-off。

* **Early Termination——done is better than perfect**

  基础算法中，若某次迭代近邻图未更新则算法终止。然而，如下图所示，近邻搜索算法随着召回率逼近 100%，需要更多次迭代才能获得略微提升，考虑到召回率和搜索性能的取舍，这些迭代其实没必要执行。为了解决这个问题，本文采取的方案是：在每次迭代中，统计近邻图更新次数 count，当 count < δKN 时终止程序。其中 δ 是精度参数，它粗略反应了由于提前终止允许错过的真正的近邻的比例。

  <img src=".\pic\NN-Descent论文浅析\glove.png" alt="glove" style="zoom: 33%;" />

* **Full Algorithm**

  > **Data**: dataset V, similarity oracle σ, K, ρ, δ // ρ 取样率、δ 精度参数
  >
  > **Result**: K-NN list B 
  >
  > begin 
  >
  > ​	B[v] ← Sample(V,K) × {<∞, true>} ∀v ∈ V 
  >
  > ​	loop 
  >
  > ​		parallel for v ∈ V do       
  >
  > ​			// 根据是否访问过将候选池分为两部分
  >
  > ​			old[v] ← all items in B[v] with a false flag // 增量搜索
  >
  > ​			new[v] ← ρK items in B[v] with a true flag  // 采样
  >
  > ​			Mark sampled items in B[v] as false;
  >
  > ​		old′ ← Reverse(old), new′ ← Reverse(new)                                        // 更新反向集
  >
  > ​		c ← 0 //update counter 
  >
  > ​		parallel for v ∈ V do                                                                              // 混合候选集和反向集
  >
  > ​			old[v] ← old[v] ∪ Sample(old′[v], ρK)  
  >
  > ​			new[v] ← new[v] ∪ Sample(new′[v], ρK) 
  >
  > ​			for $u_1, u_2$ ∈ new[v], $u_1 < u_2$ or $u_1$ ∈ new[v], $u_2$ ∈ old[v] do   // 局部连接
  >
  > ​				l ← σ($u_1, u_2$) 
  >
  > ​				// c and B[.] are synchronized.
  >
  > ​				c ← c+UpdateNN(B[$u_1$], <u2, l, true>)
  >
  > ​				c ← c+UpdateNN(B[$u_2$], <u1, l, true>)
  >
  > ​	return B if c < δNK                                                                                        // 提前终止

  


### Conclusions

* 优点：时间复杂度较低, 在大多数数据集上为 O($n^{1.14}$), 且易于 **并行化** 实现。

* 缺点：NN-Descent 在本征维度较高的数据集上表现较差，适用于本征维度较低的数据集。

  <img src=".\pic\NN-Descent论文浅析\dim.png" alt="dim" style="zoom: 50%;" />

  

### Appendix

* **理论推导**

  * 假设数据集 $V$ 半径为  $r$，数据集 $V$ 中各点 $v$ 在 $K$ 近邻图$（K=c^3）$中的 $K$ 近邻集合表示为 $B_K(v)$， $v$ 邻居的邻居的集合表示为 $B'[v]=\cup _{v'\in B[v]} B[v']$；

  * 只要近邻数量 $K$ 足够多，且满足如下三个条件：

    *  $B[v]$ 包含的 $K$ 个邻居均匀分布在 $B_r(v)$ 中;
    * 每个点分布位置相互独立;
    * $K << |B_{r/2}(v)|$;

    则即使从随机 $K$ 近邻图开始，通过探索每个点 $v$ 邻居的邻居  $B'[v]$ ，总可以找到 $v$ 的处于半径为 $r/2$ 范围内的 $K$ 近邻 $B_{r/2}(v)$；通过不断重复上述过程，每个点的邻居到该点的距离会不断收缩，最终，形成一个高质量近似 K-NNG。

  * 详细证明：

    * 若 $B_{r/2}(v)$ 中的一点 $u$，$B'[v]$ 中也包含，则至少需要存在一点 $v'$，使得 $v' \in B[v]$，且 $u \in B[v']$。接下来，只需找到满足上述条件的 $v'$（$v' \in B_{r/2}(v))$，其有以下三个不等式成立：

      * $v' \in B_r(v), P\{v' \in B[v]\} \geq K/|B_r(v)|$。（若 $v$ 的 $K$ 个邻居都在 $B_r(v)$ 中，则共有 $C^K_{|B_r(v)|}$ 种情况，而 $B_r(v)$ 中一点不是 $v$ 的邻居的情况共有 $C^K_{|B_r(v)|-1}$ 种，因此 $B_r(v)$ 中一点不是 $v$ 的邻居的概率为 $C^K_{|B_r(v)|-1} / C^K_{|B_r(v)|}$ ， 而 $B_r(v)$ 中一点是 $v$ 的邻居的概率为 $1-C^K_{|B_r(v)|-1} / C^K_{|B_r(v)|}$，即 $K/B_r(v)$）;

      * $d(u,v') \leq d(u,v) + d(v,v') \leq r,P\{u \in B[v']\} \geq K/|B_r(v')|$。（由第一条不等式可知，$B_r(v')$ 中的一点是 $v'$ 的邻居的概率为 $K/|B_r(v')|$，由于 $u$ 和 $v'$ 的距离小于等于 $r$，因此 $u$ 是 $v'$ 的邻居的概率大于等于 $K/|B_r(v')|$）；

      * $|B_r(v) \leq c|B_{r/2}(v)|, |B_r(v') \leq c|B_{r/2}(v')| \leq c|B_r(v)| \leq c^2|B_{r/2}(v)|$。（由于 $v'$ 在 $v$ 的 $r/2$ 超球中，$v'$ 的 $r/2$ 超球一定包含于 $v$ 的 $r$ 超球中）。

      <img src=".\pic\NN-Descent论文浅析\NN-Descent1.jpg" alt="NN-Descent1" style="zoom:15%;" />

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



*参考*：

* [efanna_graph](https://github.com/ZJULearning/efanna_graph)
* [KGraph: A Library for Approximate Nearest Neighbor Search](https://github.com/aaalgo/kgraph)
* [NN-Descent构建K近邻图——论文超详细注解](https://blog.csdn.net/whenever5225/article/details/105598694)

