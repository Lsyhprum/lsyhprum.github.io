---
title: ICML'19-Learning to Route in Similarity Graphs
date: 2022-04-07 09:12:46
tags:
- ICML
categories:
- [ANNS, GRAPH-BASED, ML]
mathjax: true
hidden: true
---

# TL;DR

针对基于图的方法存在的 **局部最优** 问题，本文通过学习近邻图全局结构信息，为检索提供最优的 **路由决策**。实验表明，该方法显著提高了整体搜索性能。

<!--more-->

# Background on Routing Problems

路由问题是经典的组合优化问题（COP）。对于给定的一系列节点或边，它要求以特定顺序遍历并使路径满足给定约束。这类问题属于 NP-Hard，因此，近年来以深度学习方法解决路由问题成为主流，其工作流程主要由以下 5 个步骤组成：

<img src="pipeline.png" alt="pipeline" style="zoom:80%;" />

* **Defining the problem via graphs**

  简单的路由问题是由全连接图定义的，这显然难以适用于大型问题。因此，通过启发式算法对全连接图进行稀疏化从而加速学习至关重要。

  <img src="step1.png" alt="step1" style="zoom:80%;" />

* **Obtaining latent embeddings for graph nodes and edges**

  将图中的节点或边的嵌入表示作为输入。目前常使用 GNN 获取图节点的嵌入，在每一层当中，节点从邻居收集特征，再通过递归消息传递来表示局部图结构。堆叠 L 层后，网络就能从每个节点的 L 跳邻域中构建节点的特征。其中，Transformers 和 Gated Graph ConvNets 等各向异性和基于注意力的 GNN 允许每个节点根据其对解决手头任务的相对重要性来权衡其邻居节点，它们已成为编码路由问题的默认选择。

  <img src="step2.png" alt="step2" style="zoom:80%;" />

* **Converting embeddings into discrete solutions**

  为每个节点或每条边分配属于解集的概率，然后通过经典的图搜索技术，例如概率预测引导的贪心算法或束搜索，将预测概率转换为离散决策。

  <img src="step3.png" alt="step3" style="zoom:80%;" />

* **Training the model**

  将 encoder-decoder 模型以端到端的方式进行训练，简单的情况下，可以监督学习来训练模型以产生接近最优的解。然而，为监督学习创建标记数据集是一个昂贵且耗时的过程，特别是对于大规模问题实例，最佳求解器在准确性上的保证可能不复存在，并导致用于监督训练的解决方案不精确。因此，在缺乏标准解决方案的情况下，使用强化学习解决路由问题通常是一种优雅的替代方案。



# Algorithms

* **Stochastic Search Model**

  本文将路由过程看成一个概率模型，使用学习嵌入的负内积替换原始数据空间中的距离，并根据当前堆 H 中节点的 Softmax 概率分布而非距离选取下一跳节点：

  $$P(v_i|q,H, \theta)=\frac{e^{<f_{\theta}(v_i),g_{\theta}(q)>}}{\sum_{v_j \in H} e^{<f_{\theta}(v_j),g_{\theta}(q)>} }$$

  其中，$f_{\theta}(\cdot)$ 是一个用于数据库点的神经网络，而 $g_{\theta}(\cdot)$ 是另一个用于查询的神经网络，它们的参数能够帮助随机搜索到达最近邻并将其作为 k 个输出点之一返回。

* **Optimal routing**

  假设 $\mathcal{Ref}(H)$ 从堆 $H$ 中选取最接近实际最近邻 $v^*$ 的节点，则搜索过程中每一跳由 $\mathcal{Ref}(H)$ 选取的节点序列构成最优路由。

  在实际训练中，$\mathcal{Ref}(H)$ 通过 BFS 算法获取。此外，为了提高训练速度，可能涉及的距离计算（和最小跳数差小于 5）被保存在缓存中。

* **Training objective**

  训练最佳路由的一种简单的方法（teacher forcing）是最大化对数可能性：

  $J_{naive}=E_{q,v^*}logP(Opt(q)|q,\theta)=E_{q,v^*}\sum_{v_i,H_i \in Opt(q)}logP(v_i|q,H_i,\theta)+logP(v^* \in TopK|q, V, \theta)$

  其中，$v_i,H_i \in Opt(q)$ 表示 $q$ 的最优路径上每一步的迭代顶点和堆状态，$P(v^* \in TopK|q, V, \theta)$ 是 $v*$ 被选为 top-K 之一的概率，如果 $v*$ 属于前 $k$ 个，则整体搜索成功。
  
  然而以这种方法不适用与数据库外查询，一旦该搜索算法选择非最佳节点，它会将错误的节点添加到堆 $H$中，搜索算法将进入训练期间从未出现过的状态。换句话说，上述算法没有经过训练以补偿路由错误。为了缓解该问题，该论文采用了模仿学习（imitation learning）的方式，允许选择次优节点。损失函数表示如下：
  
  $J_{naive}=E_{q,v^*}\sum_{v_i,H_i \in Search_{\theta}(q)}logP(v_i \in Ref(H_i)|q,H_i,\theta)+logP(v^* \in TopK|q, V, \theta)$
  
  <img src="compare.png" alt="compare" style="zoom:80%;" />



# Appendix

* **代码实现**

  * **相关实现类**
  
    ```python
    """ 图属性数据类 """
    class Graph:
        # ...
       
    """ 路由算法 """    
    class HNSW:
        # ...
    
    """ 模仿学习智能体，数据映射模型 """        
    class GCNWalkerAgent(SimpleWalkerAgent):
        # ...
            
    """ 获取最优路径 """            
    class ParallelBFS:
        # ...
    
    """ 训练封装类 """        
    class SupervisedWalkerTrainer:
        # ...
    ```
  
  * **训练流程**
  
    ```python
    # 生成测试数据
    # ...
    
    # 生成训练路由数据
    def generate_training_batches(...):
        # 数据分批
        for chunk_queries, chunk_initial_vertices, chunk_gt in iterate_minibatches():
            # 获取嵌入表示的搜索路径记录
            records = self.generate_records(...)
    
            # 满足一定数量，输出批数据
            # ...
    
    """ 生成路由记录 """
    def generate_records(...):
            # 计算数据集和查询集对应的嵌入表示
            query_vectors, vertex_id_to_vectors = prepare_vectors(agent, hnsw.graph, queries, ...)
    
            # 1. 获取嵌入表示路由路径
            make_sample_job = joblib.delayed(_sample_job)
            samples_jobs = (make_sample_job(*args, **kwargs)
                            for args in zip(query_vectors, initial_vertices, ground_truth))
            trajectories = joblib.Parallel(n_jobs=n_jobs, timeout=timeout, backend='multiprocessing')(samples_jobs)
    
            # 2. 封装路由过程中访问过的节点，并计算真实距离
            visited_ids = []
            for record in trajectories:
                all_vertices = list(record['visited_ids'] | {
                    record['initial_vertex_id'], record['ground_truth_id']})
                visited_ids.append(all_vertices)
    		# list of [{vertex id -> dist}]
            oracle_distances = oracle(ground_truth, visited_ids, n_jobs=n_jobs)
            
            # 3. 计算每个决策的权重
            objectives = [_supervision_job(*args)
                          for args in zip(queries.data.numpy(), trajectories, oracle_distances)]
    
            # Note: using joblib twice sometimes leads to memory leak. So we comment it for stability.
            # make_supervision_job = joblib.delayed(_supervision_job)
            # supervision_jobs = (make_supervision_job(*args, **kwargs)
            #         for args in zip(queries.data.numpy(), trajectories, oracle_distances))
            # objectives = parallel_pool(supervision_jobs)
            return objectives
            
    def _sample_job(query_vec, initial_vertex_id, ground_truth_id, **kwargs):
        # ....
        return sample_agent_trajectory(...)
    
    def _supervision_job(query, recorded_trajectory, vertex_id_to_dist, **kwargs):
        # ...
        return compute_expert_supervision(...)        
            
    train_batcher = trainer.generate_training_batches(
        graph.train_queries, graph.train_gt,
        queries_per_chunk=sampling_chunk_size, batch_size_max=batch_size,
        sampling_temperature=sampling_temperature, n_jobs=n_jobs,
    )
    
    def train_on_batch(self, batch_records, prefix='train/', **kwargs):
        """ Trains on records sampled from self.generate_training_batches """
        start_time = time.time()
        learning_rate = self.update_learning_rate(self.step, **self.learning_rate_opts)
        batch_metrics = self.compute_metrics_on_batch(batch_records, **kwargs)
        batch_metrics['loss'].backward()
        self.opt.step()
        self.opt.zero_grad()
    
        batch_metrics['learning_rate'] = learning_rate
        batch_metrics['records_per_batch'] = len(batch_records)
        batch_metrics['step_time'] = time.time() - start_time
        for key, value in batch_metrics.items():
            self.writer.add_scalar(prefix + key, value, global_step=self.step)
        self.step += 1
        return batch_metrics
    
    for batch_records in train_batcher:
        # 基于决策记录训练
        metrics_t = trainer.train_on_batch(batch_records)
        ...
    ```
  
  * **获取检索路径**
  
    ```python
    def sample_agent_trajectory(hnsw, query_vector, vertex_id_to_vectors, *,
                                initial_vertex_id, ground_truth_id, terminate_on_gt=False,
                                sampling_temperature=1.0, eps=1e-6, **kwargs):
        # ...
    
        candidates = []  # heap of vertices from smallest predicted distance to largest
        ef_top = []  # heap of top-ef vertices from largest predicted distance to smallest. Used for pruning
    
        while len(candidates) != 0:
            # record which candidates were available for selection (for loss)
            if not found_gt:
                training_history.append(dict(name='select_next', found_gt=found_gt,
                                             candidate_vertices=[v for _, v in candidates]))
    
            # 1. pop vertex according to graph walker
            estimated_distance, vertex_id = heappop(candidates)
    
            # 2. gather all next vertices
            neighbor_ids = [neighbor_id for neighbor_id in hnsw.graph.edges[vertex_id]
                            if neighbor_id not in visited_ids]
            visited_ids.update(neighbor_ids)
    
            # 3. compute distances and add all neighbors to candidates
            if len(neighbor_ids) > 0:
                neighbor_vectors = np.stack([vertex_id_to_vectors[ix] for ix in neighbor_ids])
                distances = hnsw.distance_for_routing(query_vector[None], neighbor_vectors)
            else:
                distances = []
    
            # add penalty for pruning neighbors from heap by lower bound
            # this penalty will only be applied to edges that are closer to gt than best vertex in heap
            penalty_for_pruning = dict(name='do_not_prune_optimal', neighbors=[], lower_bounds=[])
            had_gt_before = found_gt
    		
            # 更新候选集
            for distance, neighbor_id in zip(map(float, distances), neighbor_ids):
                # 获取吊车尾
                if len(ef_top) > hnsw.ef:
                    penalty_for_pruning['neighbors'].append(neighbor_id)
                    penalty_for_pruning['lower_bounds'].append(lower_bound_vertex_id)
    
                if sampling_temperature != 0:
                    noise = -np.log(-np.log(np.random.uniform(0.0, 1.0) + eps) + eps)
                    distance -= noise * sampling_temperature
    			
                if distance < lower_bound_distance or len(ef_top) < hnsw.ef:
                    heappush(candidates, (distance, neighbor_id))
                    heappush(ef_top, (-distance, neighbor_id))
    
                    if len(ef_top) > hnsw.ef:
                        _, pruned_vertex_id = heappop(ef_top)
    
                        # loss: make sure we didn't prune gt vertex
                        if pruned_vertex_id == ground_truth_id:
                            training_history.append(dict(
                                name='keep_gt_in_heap',
                                positive_ids=[pruned_vertex_id],
                                negative_ids=[nsmallest(1, ef_top)[0][1]],
                            ))
    
                    neg_lower_bound_distance, lower_bound_vertex_id = nsmallest(1, ef_top)[0]
                    lower_bound_distance = -neg_lower_bound_distance
    
                    # if we found gt, we may want to terminate
                    found_gt = found_gt or (neighbor_id == ground_truth_id)
                    if found_gt and terminate_on_gt: break
    
                else:
                    pass  # pruned by lower bound
    
                # early stopping by dcs
                dcs += 1
                if dcs >= hnsw.max_dcs: break
    
            if not had_gt_before and len(penalty_for_pruning['neighbors']) > 0:
                training_history.append(penalty_for_pruning)
    
            num_hops += 1
            if num_hops >= hnsw.max_hops: break
            if dcs >= hnsw.max_dcs: break
            if found_gt and terminate_on_gt: break
    
        # make sure gt is in top-k of the resulting structure
        verification_top = nlargest(hnsw.top_vertices_for_verification, ef_top)
        vertices_for_verification = [chosen_vertex_id for _neg_distance, chosen_vertex_id in verification_top]
        vertices_for_verification = set(vertices_for_verification)
        non_top_vertices = set(v for _, v in ef_top if v not in vertices_for_verification)
    
        training_history.append(dict(
            name='select_gt', k=hnsw.top_vertices_for_verification, ef_top_heap=ef_top, gt=ground_truth_id,
            top_vertices=list(vertices_for_verification), non_top_vertices=list(non_top_vertices)
        ))
        return dict(dcs=dcs, num_hops=num_hops, training_history=training_history, found_gt=found_gt,
                    initial_vertex_id=initial_vertex_id, ground_truth_id=ground_truth_id,
                    visited_ids=visited_ids)
    ```
    
  * **计算权重**
  
    ```python
    def compute_expert_supervision(hnsw, query, record, vertex_id_to_dist,
                                   verification_ratio=0.5, select_best_visited=True, **kwargs):
        """
        takes training record as generated by sample_training_record,
        computes a list of training objectives, each defined as a sum of components:
            weight \cdot (\log{ sum_{v_i \in positive_ids} exp(-d(q, v_i)) } -
              - \log{ sum_{v_j \in concat(positive_ids, negative_ids) } exp(-d(q, v_k)) })
        Such objective essentially computes logp(any positive > all negatives) under gumbel noise
    
        :param vertex_id_to_dist: a dict {vertex id -> distance to gt} for all vertex ids in record['visited_ids']
            also necessarily including v0 and gt
        :param verify_only_if_visited_gt: if True, adds verification loss only if target vertex is visited
        :param verification_ratio: the total weight of all verification objectives equals this
            (whereas the total weight of all other objectives equals 1 - verification_ratio)
        :returns: dict(query, objectives) where objectives is a list of dictionaries,
            each element is a dictionary containing {
            'name': tag for ease of debugging
            'positive_ids': a list of vertex ids to be included in numerator AND denominator
            'negative ids': a list of vertex ids to be included in denominator only
            'weight': multiplicative coefficient
        }
        """
    
        # compute all objectives
        objectives = []
        for event in record['training_history']:
            if event['name'] == 'select_next':
                # At each step learn to select an optimal vertex from heap
                # an optimal vertex is one of the vertices that have minimal distance to gt in terms of hops
                if event['found_gt']: continue
                candidate_vertices = event['candidate_vertices']
                candidate_distances = [vertex_id_to_dist[v] for v in candidate_vertices]
                optimal_distance = min(candidate_distances)
                objectives.append(dict(
                    name=event['name'],
                    positive_ids=[v for v, d in zip(candidate_vertices, candidate_distances)
                                  if d <= optimal_distance],
                    negative_ids=[v for v, d in zip(candidate_vertices, candidate_distances)
                                  if d > optimal_distance],
                ))
            elif event['name'] == 'select_gt':
                # make sure ground_truth_id is one of the vertices returned for verification
                # if it isn't found, make
                candidate_vertices = event['top_vertices'] + event['non_top_vertices']
                candidate_distances = [vertex_id_to_dist[v] for v in candidate_vertices]
                if select_best_visited:
                    best_visited_disance = min(candidate_distances)
                    positive_ids = {v for v, d in zip(candidate_vertices, candidate_distances)
                                    if d <= best_visited_disance}
                else:
                    positive_ids = {event['gt']}
    
                negative_ids = {v for v in event['non_top_vertices'] if v not in positive_ids}
    
                objectives.append(dict(
                    name=event['name'],
                    positive_ids=list(positive_ids),
                    negative_ids=list(negative_ids),
                ))
            else:
                raise NotImplementedError("Objective '{}' isn't implemented, make sure ef == inf"
                                          .format(event['name']))
    
        assert len(objectives) == len(record['training_history'])
    
        # compute weights s.t. all verification weights add up to verification_ratio
        # and all non-verification loss adds up to (1.0 - verification_ratio)
    
        verification_rows = [row for row in objectives if row['name'] == 'select_gt']
        routing_weight = 1. / max(1, len(objectives) - len(verification_rows)) * (1 - verification_ratio)
        verification_weight = 1. / max(1, len(verification_rows)) * verification_ratio
        for row in objectives:
            row['weight'] = verification_weight if ('gt' in row['name']) else routing_weight
        return dict(query=query, objectives=objectives)
    ```
    
  * **更新损失**
  
    ```python
    def compute_metrics_on_batch(self, batch_records, **kwargs):
            # step 1: compute vectors for queries
            state = self.agent.prepare_state(self.hnsw.graph, device=self.device, **kwargs)
            batch_queries = torch.tensor([rec['query'] for rec in batch_records], device=self.device)
            batch_query_vectors = self.agent.get_query_vectors(batch_queries, state=state,
                                                               device=self.device, **kwargs)
    
            # step 2: precompute all edge vectors, deduplicate for efficiency
            all_vertices = set()
            vertices_per_record = []
            for rec in batch_records:
                rec_edges = set()
                for row in rec['objectives']:
                    rec_edges.update(row['positive_ids'])
                    rec_edges.update(row['negative_ids'])
                all_vertices.update(rec_edges)
                vertices_per_record.append(rec_edges)
    
            all_vertices = list(all_vertices)
            vertex_to_ix = {edge: i for i, edge in enumerate(all_vertices)}
    
            batch_vertex_vectors = self.agent.get_vertex_vectors(
                all_vertices, state=state, device=self.device, **kwargs)
    
            # compute all distances
            query_ix, vertex_ix, is_numerator, row_ix, col_ix, row_weights = [], [], [], [], [], []
            row_index = 0
            for i, rec in enumerate(batch_records):
                for row in rec['objectives']:
                    num_vertices = (len(row['positive_ids']) + len(row['negative_ids']))
                    query_ix.extend([i] * num_vertices)
                    vertex_ix.extend([vertex_to_ix[vertex_id] for vertex_id in row['positive_ids'] + row['negative_ids']])
                    is_numerator.extend([1] * len(row['positive_ids']) + [0] * len(row['negative_ids']))
                    row_ix.extend([row_index] * num_vertices)
                    col_ix.extend(range(num_vertices))
                    row_weights.append(row.get('weight', 1.0))
                    row_index += 1
    
            distances = self.hnsw.distance_for_routing(batch_query_vectors[query_ix], batch_vertex_vectors[vertex_ix])
            is_numerator = torch.tensor(is_numerator, dtype=torch.uint8, device=distances.device)
            row_ix = torch.tensor(row_ix, dtype=torch.int64, device=distances.device)
            col_ix = torch.tensor(col_ix, dtype=torch.int64, device=distances.device)
            row_weights = torch.tensor(row_weights, dtype=torch.float32, device=distances.device)
    
            logits = -distances
    
            # construct two matrices, both of shape [num_training_instances, num_vertices_per_instance]
            # first matrix contains all logits, padded by -inf
            all_logits_matrix = torch.full([row_index, col_ix.max() + 1], -1e9, device=logits.device)
            all_logits_matrix[row_ix, col_ix] = logits
    
            # second matrix only contains reference logits only
            ref_logits_matrix = torch.full([row_index, col_ix.max() + 1], -1e9, device=logits.device)
            ref_logits_matrix[row_ix, col_ix] = torch.where(is_numerator, logits, torch.full_like(logits, -1e9))
    
            logp_any_ref = torch.logsumexp(ref_logits_matrix, dim=1) \
                           - torch.logsumexp(all_logits_matrix, dim=1)
    
            xent = -torch.sum(logp_any_ref * row_weights) / torch.sum(row_weights)
            acc = torch.sum(torch.ge(torch.exp(logp_any_ref), 0.5).to(dtype=torch.float32)
                            * row_weights) / torch.sum(row_weights)
            return dict(loss=xent, xent=xent, acc=acc)
    ```
  
  
  
  



***

*Bibliography*：

**Title**: [Learning to Route in Similarity Graphs.](http://proceedings.mlr.press/v97/baranchuk19a/baranchuk19a.pdf)

**Affiliations**: *Yandex*

**Journal**: *ICML'2019*

<br/>

*References*：

* [Learning to route in similarity graphs](https://github.com/dbaranchuk/learning-to-route)
* [Recent Advances in Deep Learning for Routing Problems](https://www.chaitjo.com/post/deep-learning-for-routing-problems/?continueFlag=b220d49bda26d4033730216fbc9275d5)















