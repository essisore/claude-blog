---
layout: ../../layouts/BlogPost.astro
title: "HNSW 索引实现（faiss）"
description: "学习 faiss HNSW 索引的笔记"
publishDate: "2025-10-22"
tags: ["ai", "vector", "hnsw"]
---

HNSW（Hierarchical Navigable Small World，分层可导航小世界图）是向量搜索领域中最基础和常用的一种索引结构。

原始 [论文](https://arxiv.org/pdf/1603.09320) 最初于2016年提交到 arXiv。

网上讲解 HNSW 原理的文章有很多，对这篇论文的介绍也有不少，本文不再赘述。
本文的主要记录我所学习到的 faiss 当中 HNSWFlat 索引的实现细节。


HNSW 是一个纯内存分层图结构：每一层都是一个近邻图，其中顶点代表向量，边代表向量之间的邻近关系。最底层是第0层，越往上层数越高，边的平均长度也会越大。第 0 层包含所有的向量，越往上，节点数量呈指数减少。查询时，从最高层开始，到第 1 层为止，快速找到与查询向量 q 最近的向量 v，再以 v 为入口，在第 0 层进行搜索。

## 编译

```bash
sudo apt install libopenblas-dev # 线性代数库

cmake -B build \
    -DCMAKE_BUILD_TYPE=Debug \
    -DFAISS_ENABLE_GPU=OFF \
    -DFAISS_ENABLE_PYTHON=OFF \
    -DBUILD_TESTING=OFF \
    -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

cmake --build build -j$(nproc)
```

## 数据结构

faiss 中保存 HNSW 图的数据结构如下所示：

```cpp
struct HNSW {
    // 概率数组，记录节点落到每一层的概率，总和为 1
    // 只与 m 相关，下文中会详细说明其计算方法
    std::vector<double> assign_probas;

    // 累加邻居和，记录每一层每个节点的邻居数目，只不过是一个累加值
    // 用于快速定位向量最某一层的邻居
    std::vector<int> cum_nneighbor_per_level;

    // 每个向量的最高层数，从 1 开始计数，与向量一对一
    std::vector<int> levels;

    // offsets 保存的是每个向量的邻居信息在 neighbors 数组中的偏移量，
    // 用于快速定位一个向量的邻居
    std::vector<size_t> offsets;

    // 邻居数组
    MaybeOwnedVector<storage_idx_t> neighbors;

    // 查询入口点
    storage_idx_t entry_point = -1;

    // HNSW 的最高层数
    int max_level = -1;
};
```

其本质是一个超大数组 neighbors，保存所有向量的邻居信息，如下所示：

![neighbors示例](../assets/images/faiss-hnsw-neighbors.svg)

整个邻居关系图铺平到了一维空间，前若干个元素是向量 0 的邻居，接下来是向量 1 的邻居，依此类推。每个向量的内部存储了具体每个层次的邻居，逻辑是一样的。假设向量 0 最大层次是 Level3，那么它的邻居个数是 5m 个，Level0 占 2m 个，其余层次占 m 个，按层次依次存储。

通过保存的 `offsets` 和 `cum_nneighbor_per_level`，我们可以快速定位到一个给定向量在某个层次的所有邻居，如下所示：

```cpp
// 查询给定向量在某一层的所有邻居节点信息
void HNSW::neighbor_range(
    idx_t no, // 入参： 向量编号
    int layer_no, // 入参： 层
    size_t* begin, // 出参： 邻居信息在 neighbors 的起始位置
    size_t* end, // 出参： 邻居信息在 neighbors 的结束位置
) const {
    size_t o = offsets[no];
    *begin = o + cum_nb_neighbors(layer_no);
    *end = o + cum_nb_neighbors(layer_no + 1);
}
```


下面我会以 `IndexHNSWFlat` 为例，介绍围绕 neighbors 的索引构建和查询的过程。

## 初始化

如下所示，通过定义一个 IndexHNSWFlat 对象，我们得到一个 faiss 索引，它是纯内存的：

```cpp
faiss::IndexHNSWFlat index(d, 32);
```

IndexHNSWFlat 构造函数接收三个参数，最后一个有默认值：
1. 向量维度
2. 每个节点的邻居个数 m
3. 向量距离的度量方式，默认为欧式距离

#### 计算 assign_probas

下面这段代码描述了每一层的概率是如何计算的：
```cpp
void HNSW::set_default_probas(int M, float levelMult) {
    int nn = 0;
    cum_nneighbor_per_level.push_back(0);
    for (int level = 0;; level++) {
        float proba = exp(-level / levelMult) * (1 - exp(-1 / levelMult));
        if (proba < 1e-9)
            break;
        assign_probas.push_back(proba);
        nn += level == 0 ? M * 2 : M;
        cum_nneighbor_per_level.push_back(nn);
    }
}
```

核心计算公式：

$$
P = e^{-\frac{level}{levelMult}} \times \left(1 - e^{-\frac{1}{levelMult}}\right)
$$

- 当 level 增加时，这个值指数衰减（从 1 逐渐趋向 0）,`levelMult` 控制衰减速度（越大衰减越慢）, faiss 使用的是 `1.0 / log(M)`


- `(1 - exp(-1 / levelMult)` 是归一化因子，是一个常数层，以做到让所有层的概率之和无限趋近于 1。第一部分是定义了一个等比数列,公比是 $r = e^{-\frac{1}{levelMult}}$，它的和是 $\frac{1}{1 - e^{-\frac{1}{levelMult}}}$，等比数列的每一项乘以归一化因子，等比数列的和为 1

这实际上是**几何分布**的概率质量函数（PMF）：

$$
P(level = k) = (1-p)^k \cdot p
$$

其中成功概率为：
- $p = 1 - e^{-1/levelMult}$

另外也可以看到：HNSW 第 0 层的向量的邻居个数是 2M。

当 M = 32 时，hnsw 会生成 5 层，每一层的概率为：
```
Level 0: ████████████████████████████████████████████████   96.88%
Level 1: █                                                 3.03%
Level 2:                                                   0.09%
Level 3:                                                   0.003%
Level 4:                                                   0.0001%
Level 5:                                                   0.000003%
```

可以看到，约 96.88% 的节点只会在第 0 层出现，只有约 3.12% 的节点会被选出来做导航节点。需要注意的是，这个概率是用来计算节点最高层次的，位于高层的节点，必然存在于下层节点。

## 构建

影响构建阶段的参数有两个：
- `M`：图结构中每个向量的最大的邻居个数，默认为 32
- `efConstruction`：为了找到 M 个离当前向量最近的向量，最多可以探测的向量个数，默认为 40

调用索引的 add 接口，添加一批向量数据，如下所示：

```cpp
// nb: 本次添加的向量个数
// xb：向量数据
index.add(nb, xb);
```

该接口的执行流程大致如下所示：

![HNSW 构建流程图](../assets/images/faiss-hnsw-construction.svg)

1. `prepare_level_tab`：计算每个向量的最高层次记为 `pt_level`，对 `heighbors` 数组进行扩容
2. 初始化锁，faiss 使用细粒度锁，即每个待添加的向量都会分配一把，用户防止并发更新 heighbors 数组
3. 按层次进行排序
4. 从最高层开始到最底层，按层次添加节点，当添加的节点数超过 100 时会启用并发逻辑。并发节点级别的，对于每个节点执行下面的逻辑：
    1. 获取 `entry_point`，第一个添加到图结构的点会成为入口点
    2. 加锁，保证其他线程不会更新当前节点的邻居
    3. 在当前节点的上层图结构中，使用贪心算法找出一个离当前节点最近的节点：具体而言，从 `max_level` 到 `pt_level + 1`，对于当前层应用贪心，找出该层离当前节点最近的节点 q，并以 q 为 entry point 继续在下一层中找。找的过程中维护一个 global 的值，记录离当前节点最近的节点
    4. 从第 `pt_level` 层到第 0 层，在每一层图结构中添加当前节点,更新图结构
    5. 如果 `pt_level` 大于 `max_level` 则更新 `entry_point` 和 `max_level`

### 分配 level

对于每一个将会添加到 hnsw 索引的向量，都会先为其分配一个 `pt_level`，表明它在 hnsw 中的最高层数，过程如下：

为每个向量生成一个 0 到 1 之间的随机值，根据 `assign_probas` 中的数值，来决定向量最终会落到最高层。举例（M=32 时）：当生成的随机值为 0.9，那么它会落到第 0 层；如果生成的随机值是 0.97，那么它会落到第 1 层；如果生成的随机值是 0.999，那么它会落到第 2 层

### bucket sort

这段代码位于 `hnsw_add_vertices`中，并没有抽象成一个函数，看懂了逻辑很简单：
1. 计算每个层次有多少个元素
2. offsets 每个层次中，下一个元素的位置
3. 根据向量的层次，将其填入 orders 中

用通俗地话来讲，就是在 orders 中提前为每一层的元素留好位置，然后将元素填到对应的位置上，完成排序。

### 贪心算法

贪心算法的接口定义如下：

```cpp
HNSWStats greedy_update_nearest(
        const HNSW& hnsw, // 图结构
        DistanceComputer& qdis, // 距离计算器，可以计算任意向量和查询向量的距离
        int level, // 层次
        storage_idx_t& nearest, // 入参和出参： 该层离查询向量最近的点
        float& d_nearest // 入参和出参： 最近的点和查询向量之间的距离
)
```

它的作用是在某一层找到离查询向量最近的向量。入参并未显式指定查询向量，查询向量实际上保存在 `qdis` 中。

贪心算法的过程如下：
1. 以 `nearest` 为起点，找到其在 `level` 层的所有邻居
2. 计算所有邻居到查询向量的距离
4. 如果所有的邻居都比 `nearest` 远，结束贪心
5. 更新 `nearest`，并返回步骤 1


**问题1**：是否一定会收敛？

一定会。

收敛条件是：遇到某一个点，它的所有邻居离查询向量比它本身都远。贪心算法在每一步都选择当前最近邻，并沿该方向继续搜索，确保搜索路径单调逼近查询向量。

用反证法可以清楚地证明：

假设算法不收敛，即算法会无限循环下去。这意味着：

- 算法永远不会满足终止条件 prev_nearest == nearest，即每次迭代都有 prev_nearest != nearest

由于每次迭代 prev_nearest != nearest，根据算法逻辑：

- 每次都找到了新的更近的点，即每次迭代后：d_nearest 严格递减

这样我们得到一个无限长的严格递减序列：

`d0>d1>d2>d3>...>dk>...`

同时对应的节点序列：

`n0,n1,n2,n3,...,nk,...`

显然 nk 不可能是之前出现过的点（这一点也可以用反证法证明），也就是说我们需要无限多个点。

这就产生了矛盾：我们需要无限多个互不相同的节点，但图中只有有限的 N 个节点。因此假设不成立，算法必定在有限步内收敛，且最多迭代 N 次（N 为该层节点总数）。

**问题2**：什么时候找不到最优解？

![HNSW Greedy Case](../assets/images/faiss-hnsw-greedy-case.svg)

3 是入口，q 是目标，贪心算法会停在 1（因为 1 距离 q 比它所有邻居都近）， 最优解 2 没机会访问。

### 更新图结构

当我们在高层中使用贪心算法找到一个最近的向量 `nearest` 以后，我们会在底下的每一层，即 `pt_level` 到第 0 层，以 nearest 为入口，找到待插入向量的邻居，并添加边。

下面描述的内容对于每一层都是适用的。

#### 查找邻居

更新图算法的第一步，调用 `search_neighbors_to_add` 找到待插入向量在该层的邻居。

代码中使用两个优先级队列：`candidates` 和 `results`：
- candidates 维护待插入向量的候选邻居集合，是一个小根堆，堆顶保存的是距离待插入向量最近的节点
- results 是结果集，是大根堆，堆顶保存的是距离待插入向量最远的节点


算法启动：将 nearest 添加到候选集和结果集中

算法终止（满足任一条件）：
1. 候选集为空
2. 当 candidates 中的最近的节点比 results 中最远的节点还要远时


更新候选集，步骤如下：

1. 取候选集堆顶，也就是目前距离待插入向量最近的节点
2. 从候选集中弹出堆顶元素
3. 遍历被弹出堆顶节点的所有邻居，将满足条件的邻居添加到候选果和结果集中，已经访问过的节点不会重复访问

更新候选集的核心代码如下：

```cpp
auto update_with_candidate = [&](const storage_idx_t idx,
                                    const float dis) {
    if (results.size() < hnsw.efConstruction ||
        results.top().d > dis) {
        results.emplace(dis, idx);
        candidates.emplace(dis, idx);
        if (results.size() > hnsw.efConstruction) {
            results.pop();
        }
    }
};
```

从代码可以看到：
1. 结果集的容量为 `efConstruction`
2. 当结果集未满时，节点会被添加到候选集和结果集中
3. 当结果集满了以后，只有更近的节点才会被添加到候选集和结果集中

关于算法终止条件2的思考:

1. 只有在结果集容量满了以后才会触发
2. 可能找不到最优解

从更新候选集的过程中可以看到，当结果集容量未满之前，候选集是结果集的子集，所以也就不会出现终止条件2。“可能找不到最优解”也很好理解，因为某些最优解需要通过候选集里面的邻居访问，但是这些邻居离待插入的向量又比较远，如下所示：

![HNSW Greedy Case](../assets/images/faiss-hnsw-greedy-case2.svg)

由于节点4 离 q 比 6，7 都远，所以最优解节点 2 也就没有机会被访问到了。

算法运行状态如下表所示：

| 轮次 | 候选集 | 结果集（容量为2） | 说明 |
|---|---|---|---|
|1| 3 | 3 | 算法启动时，将入口点 3 同时加入候选集和结果集 |
|2| 5 | 3, 5 | 访问 3 的邻居 5 |
|3| 1 | 5, 1| 访问 5 的邻居 1 |
|4.1 | 4 | 1, 4 | 访问 1 的邻居 4 |
|4.2 | 4, 6 | 4, 6 | 访问 1 的邻居 6 |
|4.3 | 4, 6, 7 | 6, 7 | 访问 1 的邻居 7 |
|5| 4, 6 | 6, 7 | 节点 7 没有邻居 |
|6| 4 | 6, 7 | 节点 6 没有邻居 |
| 7 | 4 | 6, 7 | 4 离 q 比 6 离 q 更远，满足条件2，退出 |


#### 邻居缩容

更新图算法第二步，调用 `::faiss::shrink_neighbor_list` 对候选邻居进行缩容。

上一步中，我们会找到 `efConstruction` 个邻居，但是每个节点最多有 `M` 个邻居，因此需要对结果进行缩容，满足 `M` 的要求。

缩容的核心逻辑：尽量找出具有代表性的邻居，而非简单的离待插入向量最近的节点。

首先会对所有邻居构建一个最小堆，按照从近到远的方式访问。一个节点会被选出来的核心逻辑： 它需要比所有已选出来的节点距离 q 更近。

如下图所示：

![HNSW Shrink Case](../assets/images/faiss-hnsw-shrink-case.svg)

会按照 v1 -> v2 -> v3 -> v4 的顺序访问：
1. 对于 v1，结果集为空，v1 加入结果集
2. 对于 v2，v2 到 v1 的距离比 v2 到 q 的距离近，v2 淘汰
3. 对于 v3，v3 到 v1 的距离比 v3 到 q 的距离远，v3 加入结果集
4. 对于 v4，v4 到 v1 和 v3 的距离比 v4 到 q 的距离远，v4 加入结果集

对于这个算法，会导致选出来的邻居个数小于 `M`。所以算法接受一个 `keep_max_size_level0` 参数，用于保证选出来的邻居个数等于 `min(M, 候选邻居个数)`

#### 添加边

更新图结果算法第三步，调用 `add_link` 添加边。

由于 HNSW 采用的是有向边，所以，添加边需要分为两步：添加我到邻居的边；添加邻居到我的边，它门调用的都是 `add_link` 的方法。

在添加邻居到我的边时，可能导致邻居的边数超过 `M` 的大小限制，所以需要处理一下。

`add_link` 实际上是更新 `neighbors` 的邻居信息。`neighbors` 采用的是预分配机制，会为某个节点留足位置保存邻居信息，这些位置初始化为 -1，代表还没有邻居。当某个节点的邻居小于 M 时，可以找到第一个空位，记录邻居 id。当邻居个数已经为 M 时，会运行一次上一步 中的 `shrink` 算法。

另外需要注意的是添加边时需要加锁：需要更新哪个节点的邻居信息，就对哪个节点进行加锁。

#### entry_point 更新逻辑

当出现更高层次的时候，会将 `entry_point` 更新为对应的节点。HNSW 是统一层并行插入，所以对于 entry_point 并没有加锁保护，因为这一层的哪一个节点作为 entry point 都是可以的。

```cpp
    if (pt_level > max_level) {
        max_level = pt_level;
        entry_point = pt_id;
    }
```

#### 思考

构建是否支持并行？

不支持。`neighbors` 扩容，`entry_point` 维护都没有锁保护。

如果上述的变量都被保护起来，能否并发 add？

个人理解：理论上可行，就是不知道构建出来的图结构效果如何。faiss 内部已经实现了细粒度的锁，允许多个线程同时往同一层插入向量，所以应该是可以支持客户端并发 add 的。

## 相似度搜索

影响相似度搜索的三个参数：
- `efSearch`: 查询过程中候选集的大小
- `check_relative_distance`： 启用时，当候选集中小于当前距离 d0 的元素数量达到 efSearch 时提前终止搜索
- `bounded_queue`: 是否使用有界队列搜索

另外，还会解释一个用户参数 `k`，表示要查询多少个 k 近邻。

相似度搜索大致流程如下所示：

![HNSW 相似度搜索流程图](../assets/images/faiss-hnsw-search-flow.svg)

对于批量查询，即有多个查询向量时，faiss 内部使用多线程技术，对于每个查询向量调用 `hnsw.search` 函数。`hnsw.search` 的核心逻辑如下所述：

1. 首先会从 `max_level` 到第 1 层，调用 `greedy_update_nearest`，找到一个距离查询向量最近的节点作为入口
2. `ef` 的实际取值是 efSearch 和 k 的最小值
3. 以第一步确定的节点为入口，在第 0 层执行搜索算法，根据用户参数 `bounded_queue` 决定使用哪种查询策略
    - search_from_candidates
    - search_from_candidate_unbounded

两个搜索算法的核心逻辑一致，区别在于 `search_from_candidates` 是有界搜索: 它维护一个固定容量 k 的结果集，只保留最好的 k 个结果；`search_from_candidate_unbounded` 是无界搜索：它返回所有满足条件的候选点，不受k值限制（只受ef限制）。

下面以 `search_from_candidates` 为例介绍搜索过程。

### MinimaxHeap

`MinimaxHeap` 是带最小值删除功能的最大堆数据结构，使用二叉堆实现：

```cpp
struct MinimaxHeap {
    int n; // 堆的容量(capacity)，最多存储n个元素容量cap
    int k; // 当前堆中的元素数量(size)当前size
    int nvalid; // 有效元素数量(≤k,因为可能有被标记删除的元素)

    std::vector<storage_idx_t> ids; // 存储索引ID的数组
    std::vector<float> dis; // 存储距离值的数组

    typedef faiss::CMax<float, storage_idx_t> HC; // 比较器

    explicit MinimaxHeap(int n) : n(n), k(0), nvalid(0), ids(n), dis(n) {}

    // 插入元素
    void push(storage_idx_t i, float v);

    // 返回堆顶元素的距离值(最大距离)，即 return dis[0];
    float max() const;

    // 返回有效元素数量，即 nvalid
    int size() const;

    // 清空堆
    void clear();

    // 删除并返回堆中最小距离的元素，删除是标记删除，即对应位置 id 记为 -1
    int pop_min(float* vmin_out = nullptr);

    // 统计距离小于 thresh 的有效元素数量
    int count_below(float thresh);
}
```


### search_from_candidates

该算法使用 candidate（MinimaxHeap，容量 ef）维护待探索节点，res（MaxHeap，容量 k）维护最优结果。

流程：

1. 从候选集取出（`pop_min`）距离 q 最小的未访问向量 v0
2. 如果 `check_relative_distance` 为真，检查当候选集中小于当前距离 d0 的元素数量达到 efSearch 时提前终止搜索
2. 标记 v0 已访问，访问其所有邻居
3. 对每个未访问过的邻居：
    - 若距离小于 res 中的最远距离或 res 未满，加入 res
    - 若候选集未满或距离小于候选集的最远距离，加入候选集
4. nstep++（nstep 是已访问的候选节点数量）
5. 当 nstep > efSearch 或候选集为空时结束，否则返回步骤 1


## 总结

本文主要记录了代码核心实现，关于算法的复杂度，各参数对算法的影响，可以阅读论文。