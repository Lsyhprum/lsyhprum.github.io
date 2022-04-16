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

当前大多数 ANNS 索引难以在单节点内存中支撑十亿级别规模的数据。因此该论文设计了一种基于 **固态硬盘** 的 ANNS 系统，且满足高召回、低时延和高数据密度的要求。

<!--more-->

# Related Work

目前，针对大规模数据的近似近邻搜索索引主要包括两种：

* **倒排索引（Inverted Index）+数据压缩**。典型的方法包括 IVFADC、IMI、IVFOADC+G+P 等。虽然能够大幅降低内存开销，但由于数据是有损压缩，召回率较低。
* **分布式索引**。该类方法将数据集分为若干个不相交的子集，并分别部署到多台机器，但这种方法代价较高。

这些方法如果直接应用于 SSD 之上，性能将大幅下降。主要的限制来自于多次读取 SSD 带来的损失，因此，设计基于 SSD 驻留索引的的挑战在于如何 **减少搜索时访问 SSD 的次数**。

# Vamana

该论文提出了基于 NSG 改进的图索引 Vamana，和目前先进的图索引相比，具有更小的搜索半径，从而能够减少磁盘访问次数。

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

具体而言，Vamana 在裁边阶段通过设置参数 $\alpha$ 控制长边的数量。整体算法流程需要执行两边上述算法：第一次 $\alpha$ 定义为 1；第二次通过设置 $\alpha \geq 1$ 增加长边。和 NSG 相比，VAMANA 增加了长边，并且 VAMANA 初始图为随机图，该论文认为相比于近似 K 近邻图，随机图构建速度更快，且能构建出质量更好的图。

![diskann](D:\BLOG\source\_posts\DiskANN论文浅析\diskann.png)

如上图所示，第一行是第一遍构图过程（$\alpha$=1），第二行是第二遍构图过程（$\alpha \geq$ 1）。可以看到，经过第二遍选边，增加了很多长边。

# DiskANN



# Appendix

* **构建 DiskANN 索引**

  ```cpp
  // 构建索引入口函数，B/M 分别限制 加载最终索引 和 索引构建中 的内存开销，PQ_disk_bytes 指定 SSD 中压缩向量的字节个数。
  template<typename T>
  bool build_index(const char* dataFilePath, const char* indexFilePath,
                   const char* indexBuildParameters, diskann::Metric metric) {
    // 构建索引参数，PQ_disk_bytes 为 SSD 中压缩向量的位数
    std::cout << "Usage: " << 
              "[R](max degree)" <<
        		"[L](indexing list size, better if >= R)" <<
        		"[B](RAM limit of final index in GB)" <<
        		"[M](memory limit while indexing)" <<  
        		"[T](number of threads for indexing)" << 
        		"[PQ_disk_bytes](for very large dimensionality, use 0 for full vectors)" 
              << std::endl;
    return diskann::build_disk_index<T>(dataFilePath, indexFilePath,
                                        indexBuildParameters, metric);
  }
  
  // 根据加载数据存储介质(内存，SSD)的不同对数据进行压缩，生成两种不同长度(num_pq_chunks, disk_pq_dims)的压缩向量。
  template<typename T>
  bool build_disk_index(const char *dataFilePath, const char *indexFilePath,
                        const char *    indexBuildParameters,
                        diskann::Metric compareMetric) {
      ...
          
      _u32 disk_pq_dims = 0; // 磁盘中压缩向量字节个数
      bool use_disk_pq = false; // 磁盘中向量数据是否压缩
  	
      ...
      
      size_t num_pq_chunks =
          (size_t)(std::floor)(_u64(final_index_ram_limit / points_num)); // 内存中压缩向量字节个数
      
      ...
  	
      // 生成磁盘中保存的压缩向量编码
      if (use_disk_pq) {
          generate_pq_pivots(train_data, train_size, (uint32_t) dim, 256,
                             (uint32_t) disk_pq_dims, NUM_KMEANS_REPS,
                             disk_pq_pivots_path, false); // 生成 PQ 码本
          
          generate_pq_data_from_pivots<T>(
              data_file_to_use.c_str(), 256, (uint32_t) disk_pq_dims,
              disk_pq_pivots_path, disk_pq_compressed_vectors_path); // 将剩余数据量化
      }
  
      ....
  	
      // 生成内存中保存的压缩向量编码
      generate_pq_pivots(train_data, train_size, (uint32_t) dim, 256,
                         (uint32_t) num_pq_chunks, NUM_KMEANS_REPS,
                         pq_pivots_path, make_zero_mean);
      generate_pq_data_from_pivots<T>(data_file_to_use.c_str(), 256,
                                      (uint32_t) num_pq_chunks, pq_pivots_path,
                                      pq_compressed_vectors_path);
  	
      // 分片构造索引
      diskann::build_merged_vamana_index<T>(
          data_file_to_use.c_str(), diskann::Metric::L2, L, R, p_val,
          indexing_ram_budget, mem_index_path, medoids_path, centroids_path);
  	
      // 生成 SSD 中加载的索引和原始数据/压缩数据
      if (!use_disk_pq) {
          diskann::create_disk_layout<T>(data_file_to_use.c_str(), mem_index_path,
                                         disk_index_path);
      } else
          diskann::create_disk_layout<_u8>(disk_pq_compressed_vectors_path,
                                           mem_index_path, disk_index_path);
  
      double sample_sampling_rate = (150000.0 / points_num);
      gen_random_slice<T>(data_file_to_use.c_str(), sample_base_prefix,
                          sample_sampling_rate);
  
      std::remove(mem_index_path.c_str());
      if (use_disk_pq)
          std::remove(disk_pq_compressed_vectors_path.c_str());
  
      return true;
  }
      
  // 给定尺寸为 num_train*dim 的训练数据，将原始向量根据列划分为 num_pq_chunks 个子向量，并通过 k-means 计算子向量对应码本。最终将码本以 num_centers*dim 浮点数组的形式存储在 pq_pivots_path 中。
  int generate_pq_pivots(const float *passed_train_data, size_t num_train,
                         unsigned dim, unsigned num_centers,
                         unsigned num_pq_chunks, unsigned max_k_means_reps,
                         std::string pq_pivots_path, bool make_zero_mean) {
    ...
  
    // 使用 rearrangement 和 chunk_offsets 存储子向量(chunks)
    std::vector<uint32_t> rearrangement;
    std::vector<uint32_t> chunk_offsets;
    
    // 确定如何将原始向量划分为 num_pq_chunks 个子向量(chunks)，和普通 PQ 不同的是，此处非顺序划分
    size_t low_val = (size_t) std::floor((double) dim / (double) num_pq_chunks);
    size_t high_val = (size_t) std::ceil((double) dim / (double) num_pq_chunks);
    size_t max_num_high = dim - (low_val * num_pq_chunks);
    size_t cur_num_high = 0;				// 列数为 high_val 的 chunks 个数
    size_t cur_bin_threshold = high_val;	// 每个 chunks 中最多使用的列数
  
    std::vector<std::vector<uint32_t>> bin_to_dims(num_pq_chunks);   // chunks 对应列序号
    tsl::robin_map<uint32_t, uint32_t> dim_to_bin;	// 上个数据结构的查找表，判断当前列是否已经属于前一个 chunk
    std::vector<float>                 bin_loads(num_pq_chunks, 0);	// 计算 chunks 中心
  
    // Process dimensions not inserted by previous loop
    for (uint32_t d = 0; d < dim; d++) {
      if (dim_to_bin.find(d) != dim_to_bin.end())
        continue;
      // 寻找当前列最适合的 chunk
      auto  cur_best = num_pq_chunks + 1;
      float cur_best_load = std::numeric_limits<float>::max();
      for (uint32_t b = 0; b < num_pq_chunks; b++) {	
        if (bin_loads[b] < cur_best_load &&
            bin_to_dims[b].size() < cur_bin_threshold) {
          cur_best = b;
          cur_best_load = bin_loads[b];
        }
      }
      diskann::cout << " Pushing " << d << " into bin #: " << cur_best
                    << std::endl;
      bin_to_dims[cur_best].push_back(d);
      if (bin_to_dims[cur_best].size() == high_val) {
        cur_num_high++;
        if (cur_num_high == max_num_high)
          cur_bin_threshold = low_val;
      }
    }
  
    ...
    
    full_pivot_data.reset(new float[num_centers * dim]); // 生成每个子向量对应 K-means 质心 
  
    for (size_t i = 0; i < num_pq_chunks; i++) {
      size_t cur_chunk_size = chunk_offsets[i + 1] - chunk_offsets[i];	// 当前 chunk 对应列数
  
      std::unique_ptr<float[]> cur_pivot_data =
          std::make_unique<float[]>(num_centers * cur_chunk_size);	// 保存子向量码本
      std::unique_ptr<float[]> cur_data =
          std::make_unique<float[]>(num_train * cur_chunk_size);	// 保存子向量
      std::unique_ptr<uint32_t[]> closest_center =
          std::make_unique<uint32_t[]>(num_train);
  	
      // 计算子向量码本
  #pragma omp parallel for schedule(static, 65536)
      for (int64_t j = 0; j < (_s64) num_train; j++) {
        std::memcpy(cur_data.get() + j * cur_chunk_size,
                    train_data.get() + j * dim + chunk_offsets[i],
                    cur_chunk_size * sizeof(float));
      }
  
      kmeans::kmeanspp_selecting_pivots(cur_data.get(), num_train, cur_chunk_size,
                                        cur_pivot_data.get(), num_centers);
  
      kmeans::run_lloyds(cur_data.get(), num_train, cur_chunk_size,
                         cur_pivot_data.get(), num_centers, max_k_means_reps,
                         NULL, closest_center.get());
  
      for (uint64_t j = 0; j < num_centers; j++) {
        std::memcpy(full_pivot_data.get() + j * dim + chunk_offsets[i],
                    cur_pivot_data.get() + j * cur_chunk_size,
                    cur_chunk_size * sizeof(float));
      }
    }
  	
    ...
    return 0;
  }
  
  // 若子向量码本个数多于 256，则存储为二进制格式的 4 字节向量(uint32_t)，否则，以 uint8_t 格式存储
  template<typename T>
  int generate_pq_data_from_pivots(const std::string data_file,
                                   unsigned num_centers, unsigned num_pq_chunks,
                                   std::string pq_pivots_path,
                                   std::string pq_compressed_vectors_path) {
    ... // 加载数据
        
    std::unique_ptr<_u32[]> block_compressed_base =
        std::make_unique<_u32[]>(block_size * (_u64) num_pq_chunks);	// 压缩向量
    std::memset(block_compressed_base.get(), 0,
                block_size * (_u64) num_pq_chunks * sizeof(uint32_t));
      
    std::unique_ptr<float[]> block_data_float =
        std::make_unique<float[]>(block_size * dim); // 存储待处理子向量
       
    // 按块存取
    size_t block_size = num_points <= BLOCK_SIZE ? num_points : BLOCK_SIZE;
    size_t num_blocks = DIV_ROUND_UP(num_points, block_size);
    for (size_t block = 0; block < num_blocks; block++) {
      size_t start_id = block * block_size;
      size_t end_id = (std::min)((block + 1) * block_size, num_points);
      size_t cur_blk_size = end_id - start_id;
  	
      ...
  
      for (size_t i = 0; i < num_pq_chunks; i++) {
        size_t cur_chunk_size = chunk_offsets[i + 1] - chunk_offsets[i];
        if (cur_chunk_size == 0)
          continue;
  
        std::unique_ptr<float[]> cur_pivot_data =
            std::make_unique<float[]>(num_centers * cur_chunk_size);	// 该子空间对应码本
        std::unique_ptr<float[]> cur_data =
            std::make_unique<float[]>(cur_blk_size * cur_chunk_size);	// 第 i 个子向量
        std::unique_ptr<uint32_t[]> closest_center =
            std::make_unique<uint32_t[]>(cur_blk_size);
          
        ...
  
        math_utils::compute_closest_centers(cur_data.get(), cur_blk_size,
                                            cur_chunk_size, cur_pivot_data.get(),
                                            num_centers, 1, closest_center.get());	// 计算对应质心编号
  
  #pragma omp parallel for schedule(static, 8192)
        for (int64_t j = 0; j < (_s64) cur_blk_size; j++) {
          block_compressed_base[j * num_pq_chunks + i] = closest_center[j];
          
          // 计算压缩向量的解码向量
  #ifdef SAVE_INFLATED_PQ	
          for (uint64_t k = 0; k < cur_chunk_size; k++)
            block_inflated_base[j * dim + chunk_offsets[i] + k] =
                cur_pivot_data[closest_center[j] * cur_chunk_size + k] +
                centroid[chunk_offsets[i] + k];
  #endif
        }
      }
  	
      // 保存数据
      if (num_centers > 256) {
        compressed_file_writer.write(
            (char *) (block_compressed_base.get()),
            cur_blk_size * num_pq_chunks * sizeof(uint32_t));
      } else {
        std::unique_ptr<uint8_t[]> pVec =
            std::make_unique<uint8_t[]>(cur_blk_size * num_pq_chunks);
        diskann::convert_types<uint32_t, uint8_t>(
            block_compressed_base.get(), pVec.get(), cur_blk_size, num_pq_chunks);
        compressed_file_writer.write(
            (char *) (pVec.get()),
            cur_blk_size * num_pq_chunks * sizeof(uint8_t));
      }
  #ifdef SAVE_INFLATED_PQ
      inflated_file_writer.write((char *) (block_inflated_base.get()),
                                 cur_blk_size * dim * sizeof(float));
  #endif
      diskann::cout << ".done." << std::endl;
    }
    
    return 0;
  }
  ```

* **搜索**

  ```cpp
  template<typename T>
  int search_disk_index(int argc, char** argv) {
    // 加载查询集、码本、索引
    ...
    
    // cached_beam_search 加载最常访问的点作为缓存，并 warm up
    ...
        
    // 正式查询
    ...
  }
  
  // 查找表
  void populate_chunk_distances(const float* query_vec, float* dist_vec) {
      memset(dist_vec, 0, 256 * n_chunks * sizeof(float));
      // 分块距离计算
      for (_u64 chunk = 0; chunk < n_chunks; chunk++) {
          // sum (q-c)^2 for the dimensions associated with this chunk
          float* chunk_dists = dist_vec + (256 * chunk);
          for (_u64 j = chunk_offsets[chunk]; j < chunk_offsets[chunk + 1]; j++) {
              _u64         permuted_dim_in_query = rearrangement[j];
              const float* centers_dim_vec = tables_T + (256 * j);
              for (_u64 idx = 0; idx < 256; idx++) {
                  double diff =
                      centers_dim_vec[idx] - (query_vec[permuted_dim_in_query] -
                                              centroid[permuted_dim_in_query]); // 码本减去当前残差
                  chunk_dists[idx] += (float) (diff * diff);
              }
          }
      }
  }
  
  void aggregate_coords(const unsigned *ids, const _u64 n_ids,
                        const _u8 *all_coords, const _u64 ndims, _u8 *out) {
      for (_u64 i = 0; i < n_ids; i++) {
          memcpy(out + i * ndims, all_coords + ids[i] * ndims, ndims * sizeof(_u8));
      }
  }
  
  void pq_dist_lookup(const _u8 *pq_ids, const _u64 n_pts,
                      const _u64 pq_nchunks, const float *pq_dists,
                      float *dists_out) {
      _mm_prefetch((char *) dists_out, _MM_HINT_T0);
      _mm_prefetch((char *) pq_ids, _MM_HINT_T0);
      _mm_prefetch((char *) (pq_ids + 64), _MM_HINT_T0);
      _mm_prefetch((char *) (pq_ids + 128), _MM_HINT_T0);
      memset(dists_out, 0, n_pts * sizeof(float));
      for (_u64 chunk = 0; chunk < pq_nchunks; chunk++) {
          const float *chunk_dists = pq_dists + 256 * chunk;
          if (chunk < pq_nchunks - 1) {
              _mm_prefetch((char *) (chunk_dists + 256), _MM_HINT_T0);
          }
          for (_u64 idx = 0; idx < n_pts; idx++) {
              _u8 pq_centerid = pq_ids[pq_nchunks * idx + chunk];
              dists_out[idx] += chunk_dists[pq_centerid];
          }
      }
  }
  
  template<typename T>
  void PQFlashIndex<T>::cached_beam_search(const T *query1, const _u64 k_search,
                                           const _u64 l_search, _u64 *indices,
                                           float *     distances,
                                           const _u64  beam_width,
                                           QueryStats *stats) {
      ...
  
      // reset query
      query_scratch->reset();
  
      // scratch space to compute distances between FP32 Query and INT8 data
      float *scratch = query_scratch->aligned_scratch;
      _mm_prefetch((char *) scratch, _MM_HINT_T0);
  
      // pointers to buffers for data
      T *   data_buf = query_scratch->coord_scratch;
      _u64 &data_buf_idx = query_scratch->coord_idx;
      _mm_prefetch((char *) data_buf, _MM_HINT_T1);
  
      // sector scratch
      char *sector_scratch = query_scratch->sector_scratch;
      _u64 &sector_scratch_idx = query_scratch->sector_idx;
  
      // query <-> PQ chunk centers distances
      float *pq_dists = query_scratch->aligned_pqtable_dist_scratch;
      pq_table.populate_chunk_distances(query_float, pq_dists);
  
      // query <-> neighbor list
      float *dist_scratch = query_scratch->aligned_dist_scratch;
      _u8 *  pq_coord_scratch = query_scratch->aligned_pq_coord_scratch;
  
      // lambda to batch compute query<-> node distances in PQ space
      auto compute_dists = [this, pq_coord_scratch, pq_dists](const unsigned *ids,
                                                              const _u64 n_ids,
                                                              float *dists_out) {
          ::aggregate_coords(ids, n_ids, this->data, this->n_chunks,
                             pq_coord_scratch);
          ::pq_dist_lookup(pq_coord_scratch, n_ids, this->n_chunks, pq_dists,
                           dists_out);
      };
      Timer                 query_timer, io_timer, cpu_timer;
      std::vector<Neighbor> retset(l_search + 1);
      tsl::robin_set<_u64>  visited(4096);
  
      std::vector<Neighbor> full_retset;
      full_retset.reserve(4096);
      tsl::robin_map<_u64, T *> fp_coords;
  
      _u32                        best_medoid = 0;
      float                       best_dist = (std::numeric_limits<float>::max)();
      std::vector<SimpleNeighbor> medoid_dists;
      for (_u64 cur_m = 0; cur_m < num_medoids; cur_m++) {
          float cur_expanded_dist = dist_cmp_float->compare(
              query_float, centroid_data + aligned_dim * cur_m,
              (unsigned) aligned_dim);
          if (cur_expanded_dist < best_dist) {
              best_medoid = medoids[cur_m];
              best_dist = cur_expanded_dist;
          }
      }
  
      compute_dists(&best_medoid, 1, dist_scratch);
  
      retset[0].id = best_medoid;
      retset[0].distance = dist_scratch[0];
      retset[0].flag = true;
      visited.insert(best_medoid);
  
      unsigned cur_list_size = 1;
  
      std::sort(retset.begin(), retset.begin() + cur_list_size);
  
      unsigned cmps = 0;
      unsigned hops = 0;
      unsigned num_ios = 0;
      unsigned k = 0;
  
      // cleared every iteration
      std::vector<unsigned>                    frontier;
      std::vector<std::pair<unsigned, char *>> frontier_nhoods;
      std::vector<AlignedRead>                 frontier_read_reqs;
      std::vector<std::pair<unsigned, std::pair<unsigned, unsigned *>>>
          cached_nhoods;
  
      while (k < cur_list_size) {
          auto nk = cur_list_size;
  
          // clear iteration state
          frontier.clear();
          frontier_nhoods.clear();
          frontier_read_reqs.clear();
          cached_nhoods.clear();
          sector_scratch_idx = 0;
  
          // find new beam
          // WAS: _u64 marker = k - 1;
          _u32 marker = k;
          _u32 num_seen = 0;
  
          while (marker < cur_list_size && frontier.size() < beam_width &&
                 num_seen < beam_width + 2) {
              if (retset[marker].flag) {
                  num_seen++;
                  auto iter = nhood_cache.find(retset[marker].id);
                  if (iter != nhood_cache.end()) {
                      cached_nhoods.push_back(
                          std::make_pair(retset[marker].id, iter->second));
                      if (stats != nullptr) {
                          stats->n_cache_hits++;
                      }
                  } else {
                      frontier.push_back(retset[marker].id);
                  }
                  retset[marker].flag = false;
                  if (this->count_visited_nodes) {
                      reinterpret_cast<std::atomic<_u32> &>(
                          this->node_visit_counter[retset[marker].id].second)
                          .fetch_add(1);
                  }
              }
              marker++;
          }
  
          // read nhoods of frontier ids
          if (!frontier.empty()) {
              if (stats != nullptr)
                  stats->n_hops++;
              for (_u64 i = 0; i < frontier.size(); i++) {
                  auto                    id = frontier[i];
                  std::pair<_u32, char *> fnhood;
                  fnhood.first = id;
                  fnhood.second = sector_scratch + sector_scratch_idx * SECTOR_LEN;
                  sector_scratch_idx++;
                  frontier_nhoods.push_back(fnhood);
                  frontier_read_reqs.emplace_back(
                      NODE_SECTOR_NO(((size_t) id)) * SECTOR_LEN, SECTOR_LEN,
                      fnhood.second);
                  if (stats != nullptr) {
                      stats->n_4k++;
                      stats->n_ios++;
                  }
                  num_ios++;
              }
              io_timer.reset();
              #ifdef USE_BING_INFRA
              reader->read(frontier_read_reqs, ctx, true);  // async reader windows.
              #else
              reader->read(frontier_read_reqs, ctx);  // synchronous IO linux
              #endif
              if (stats != nullptr) {
                  stats->io_us += io_timer.elapsed();
              }
          }
  
          // process cached nhoods
          for (auto &cached_nhood : cached_nhoods) {
              auto  global_cache_iter = coord_cache.find(cached_nhood.first);
              T *   node_fp_coords_copy = global_cache_iter->second;
              float cur_expanded_dist;
              if (!use_disk_index_pq) {
                  cur_expanded_dist = dist_cmp->compare(query, node_fp_coords_copy,
                                                        (unsigned) aligned_dim);
              } else {
                  if (metric == diskann::Metric::INNER_PRODUCT)
                      cur_expanded_dist = disk_pq_table.inner_product(
                      query_float, (_u8 *) node_fp_coords_copy);
                  else
                      cur_expanded_dist = disk_pq_table.l2_distance(
                      query_float, (_u8 *) node_fp_coords_copy);
              }
              full_retset.push_back(
                  Neighbor((unsigned) cached_nhood.first, cur_expanded_dist, true));
  
              _u64      nnbrs = cached_nhood.second.first;
              unsigned *node_nbrs = cached_nhood.second.second;
  
              // compute node_nbrs <-> query dists in PQ space
              cpu_timer.reset();
              compute_dists(node_nbrs, nnbrs, dist_scratch);
              if (stats != nullptr) {
                  stats->n_cmps += nnbrs;
                  stats->cpu_us += cpu_timer.elapsed();
              }
  
              // process prefetched nhood
              for (_u64 m = 0; m < nnbrs; ++m) {
                  unsigned id = node_nbrs[m];
                  if (visited.find(id) != visited.end()) {
                      continue;
                  } else {
                      visited.insert(id);
                      cmps++;
                      float dist = dist_scratch[m];
                      // diskann::cout << "cmp: " << id << ", dist: " << dist <<
                      // std::endl; std::cerr << "dist: " << dist << std::endl;
                      if (dist >= retset[cur_list_size - 1].distance &&
                          (cur_list_size == l_search))
                          continue;
                      Neighbor nn(id, dist, true);
                      auto     r = InsertIntoPool(
                          retset.data(), cur_list_size,
                          nn);  // Return position in sorted list where nn inserted.
                      if (cur_list_size < l_search)
                          ++cur_list_size;
                      if (r < nk)
                          nk = r;  // nk logs the best position in the retset that was
                      // updated
                      // due to neighbors of n.
                  }
              }
          }
          #ifdef USE_BING_INFRA
          // process each frontier nhood - compute distances to unvisited nodes
          int completedIndex = -1;
          // If we issued read requests and if a read is complete or there are reads
          // in wait
          // state, then enter the while loop.
          while (frontier_read_reqs.size() > 0 &&
                 getNextCompletedRequest(ctx, frontier_read_reqs.size(),
                                         completedIndex)) {
              if (completedIndex == -1) {  // all reads are waiting
                  continue;
              }
              auto &frontier_nhood = frontier_nhoods[completedIndex];
              (*ctx.m_pRequestsStatus)[completedIndex] = IOContext::PROCESS_COMPLETE;
              #else
              for (auto &frontier_nhood : frontier_nhoods) {
                  #endif
                  char *node_disk_buf =
                      OFFSET_TO_NODE(frontier_nhood.second, frontier_nhood.first);
                  unsigned *node_buf = OFFSET_TO_NODE_NHOOD(node_disk_buf);
                  _u64      nnbrs = (_u64)(*node_buf);
                  T *       node_fp_coords = OFFSET_TO_NODE_COORDS(node_disk_buf);
                  //        assert(data_buf_idx < MAX_N_CMPS);
                  if (data_buf_idx == MAX_N_CMPS)
                      data_buf_idx = 0;
  
                  T *node_fp_coords_copy = data_buf + (data_buf_idx * aligned_dim);
                  data_buf_idx++;
                  memcpy(node_fp_coords_copy, node_fp_coords, disk_bytes_per_point);
  
                  float cur_expanded_dist;
                  if (!use_disk_index_pq) {
                      cur_expanded_dist = dist_cmp->compare(query, node_fp_coords_copy,
                                                            (unsigned) aligned_dim);
                  } else {
                      if (metric == diskann::Metric::INNER_PRODUCT)
                          cur_expanded_dist = disk_pq_table.inner_product(
                          query_float, (_u8 *) node_fp_coords_copy);
                      else
                          cur_expanded_dist = disk_pq_table.l2_distance(
                          query_float, (_u8 *) node_fp_coords_copy);
                  }
                  full_retset.push_back(
                      Neighbor(frontier_nhood.first, cur_expanded_dist, true));
  
                  unsigned *node_nbrs = (node_buf + 1);
                  // compute node_nbrs <-> query dist in PQ space
                  cpu_timer.reset();
                  compute_dists(node_nbrs, nnbrs, dist_scratch);
                  if (stats != nullptr) {
                      stats->n_cmps += nnbrs;
                      stats->cpu_us += cpu_timer.elapsed();
                  }
  
                  cpu_timer.reset();
                  // process prefetch-ed nhood
                  for (_u64 m = 0; m < nnbrs; ++m) {
                      unsigned id = node_nbrs[m];
                      if (visited.find(id) != visited.end()) {
                          continue;
                      } else {
                          visited.insert(id);
                          cmps++;
                          float dist = dist_scratch[m];
                          // diskann::cout << "cmp: " << id << ", dist: " << dist <<
                          // std::endl;
                          // diskann::cout << "dist: " << dist << std::endl;
                          if (stats != nullptr) {
                              stats->n_cmps++;
                          }
                          if (dist >= retset[cur_list_size - 1].distance &&
                              (cur_list_size == l_search))
                              continue;
                          Neighbor nn(id, dist, true);
                          auto     r = InsertIntoPool(
                              retset.data(), cur_list_size,
                              nn);  // Return position in sorted list where nn inserted.
                          if (cur_list_size < l_search)
                              ++cur_list_size;
                          if (r < nk)
                              nk = r;  // nk logs the best position in the retset that was
                          // updated
                          // due to neighbors of n.
                      }
                  }
  
                  if (stats != nullptr) {
                      stats->cpu_us += cpu_timer.elapsed();
                  }
              }
  
              // update best inserted position
              //
  
              if (nk <= k)
                  k = nk;  // k is the best position in retset updated in this round.
              else
                  ++k;
  
              hops++;
          }
  
          // re-sort by distance
          std::sort(full_retset.begin(), full_retset.end(),
                    [](const Neighbor &left, const Neighbor &right) {
                        return left.distance < right.distance;
                    });
  
          /*
          std::cout<<"return set: \n";
          for (auto &x : full_retset)
          std::cout<<x.id<<"\t" <<x.distance<<std::endl;
          std::cout<<std::endl;
      */
  
          // copy k_search values
          for (_u64 i = 0; i < k_search; i++) {
              indices[i] = full_retset[i].id;
              if (distances != nullptr) {
                  distances[i] = full_retset[i].distance;
                  if (metric == diskann::Metric::INNER_PRODUCT) {  // flip the sign from
                      // convert min to max
                      distances[i] = (-distances[i]);
                      if (max_base_norm != 0)
                          distances[i] *= (max_base_norm *
                                           query_norm);  // rescale to revert back to original
                      // norms (cancelling the effect of
                      // base and query pre-processing)
                  }
              }
          }
  
          this->thread_data.push(data);
          this->thread_data.push_notify_all();
  
          if (stats != nullptr) {
              stats->total_us = (double) query_timer.elapsed();
          }
      }
  ```

  

****



*Bibliography*：

**Title**: [DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node.](https://suhasjs.github.io/files/diskann_neurips19.pdf)

**Affiliations**: *Microsoft Research India*

**Journal**: *NeurIPS'2019*

**Key Words**：*High-dimensional indexing, image indexing, very large databases, approximate search.*
