---
layout: ../../layouts/BlogPost.astro
title: "理解 go 的 syncMap"
description: "介绍 go 的 syncMap 实现原理"
publishDate: "2023-07-17"
tags: ["go", "hashmap", "threads"]
---


## 概述

```go
type Map struct {
	mu Mutex
	read atomic.Pointer[readOnly] // ! readOnly
	dirty map[any]*entry
	misses int
}
```

sync.Map 很有趣，其设计思想与数据库内核里面的 MVCC 有异曲同工之妙。

如上所示，sync.Map 使用一个 `readOnly` 和 `dirty` 来实现线程安全，具体而言：

- **readOnly 持有 key 的最新数据**，提供一个 read view，读取 map 时，
会先从 readOnly 中读，如果 readOnly 没有的话，
才会去 dirty 中读。读 readOnly 是原子操作，无需加锁

- **dirty 可以理解为 readOnly 的超集**，包含所有 readOnly 的 key 以及新插入的 key（脏数据）。
读写 dirty 时需要加锁。dirty 会在一定的策略下升级为 readOnly。

sync.Map 支持并发操作，是线程安全的，但并非完全 lock free。

简要概括其主要操作：

1. 读 （Load）
    
    先从 readOnly 中读，如果 readOnly 没有的话，再从 dirty 中读，并更新 miss 数目。
    需要注意的是，不管从 dirty 中有没有读到，都认为是一次 miss。
    
2. 写（Store）
    
    如果 key 存在，直接写入 readOnly，否则需要写入 dirty 中。
    
3. 删除（Delete）
    
    采用**标记删除**，即：将 readOnly 中的 key 所对应的 entry 置空。
    如果 readOnly 中不存在该 key，则从 dirty 中直接删除。dirty 初始化时，
    被标记删除的元素不会从 readOnly 拷贝到 dirty 中；
    dirty 升级时，会替换掉 readOnly 中的所有内容，如此便完成从 readOnly 中的实际删除该元素。
    

### 适用场景

1. 读多写少
2. 很多 goroutine 并发操作不相交的 key

## 实现细节

### dirty 初始化

dirty 的初始化发生在第一次往空 dirty 中写入时，具体而言：

1. 会根据 readOnly 中未被删除的元素构造出整个 dirty，value 是一个**浅拷贝**，
即 dirty 里面的 key 和 readOnly 的 key 指向同一个 value
（P.S. 浅拷贝很关键，意味着更新 readOnly 时也更新了 dirty）。
构造过程中，同时会将 readOnly 中被标记删除的元素打上 `expunged` 标记

2. 设置 readOnly 的 amended 属性，该属性表示 readOnly 中的数据不完整，
即 dirty 中包含一些 readOnly 没有的 key

### dirty 升级

每次更新 miss 数目的时候，都要检查一下是否需要升级。
具体而言：当 miss 数目追上 dirty 中元素的个数时，便会触发一次升级操作。升级操作分为三步：

1. 将 dirty 升级为 readOnly，并清除 readOnly 的 amended 属性
2. 置空 dirty
3. 置零 miss 数目

### 读 Map

1. 首先从 readOnly 中读，如果存在，还要判断 value 是否为空或者是否被标记删除了？
如果被删除了，返回 key 不存在，否则返回对应的 value

2. 没读到且 readOnly 已经是最新的，即 amended 为 false，返回 key 不存在

3. 没读到，但是 dirty 中可能有。加锁，读 dirty，并更新一下 miss 数目，解锁。
这其中还会重新读一遍 readOnly，主要是防止加锁过程中，dirty 发生了升级

### 从 Map 中删除元素

删除 key 的前提是找到 key，所以删除其实跟读操作类似。

1. 从 readOnly 中读取，如果存在，对读取到的 old_value 进行标记删除。
具体而言，就是将 readOnly 该 key 的 value 置空。如果已经被标记或者是已经为空，也就不需要操作了

2. 如果 readOnly 中没有，但 dirty 中可能有？直接从 dirty 中删除 key，并更新一下 miss 数目

### 写 Map

现在要往 map 中写入键值对 (key, new_value)，因为 key 可能已经存在，我们使用 new_value 区分 old_value。

1. 如果 readOnly 存在该 key，说明以前插入过该 key。
    - 如果 key 没有被标记删除，直接调用 `CompareAndSwap` 将 new_value 写进去
    - 如果发现 key 已经被标记为删除，那么就需要往 dirty 里面写

2. 如果 readOnly 中不存在该 key，抑或是存在但标记删除，我们就需要往 dirty 里面写。写 dirty 之前需要加锁。
为了防止在读 readOnly 和加锁之间发生了 dirty 升级，需要在加锁之后重新读一遍 readOnly：
    - 第一种情况，再次读取 readOnly 后读取到了 key，说明其他线程写入了 key，并触发了 dirty 升级。
    在这种情况下，首先需要判断是否有 expunged 标记：
        - 如果被标记删除，我们需要同时更新 readOnly 和 dirty，将 (key, new_value) 写进去
        - 如果没有被标记删除，那么我们只需要更新 readOnly 即可

    - 第二种情况，再次读取 readOnly 后并没有读到 key，但是 dirty 中存在 key。
    此时，直接用 new_value 更新 dirty。

    - 第三种情况，没有读到，且 dirty 中也不存在（P.S. 意味着插入了一个从来没有插入过的 key）。
    此时直接将 new_value 插入到 dirty 中，但在插入之前，需要初始化 dirty。
