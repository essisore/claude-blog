---
layout: ../../layouts/BlogPost.astro
title: "KMemStoreDB Undo 日志与事务可见性"
description: "介绍 KMemStoreDB 内存引擎 Undo 日志实现与事务可见性判断"
publishDate: "2025-07-12"
tags: ["数据库内核原理", "事务", "undo"]
---

本文介绍我们在 KMemStoreDB（简称：kmstore） 内存引擎中实现的 undo 日志以及基于 undo 日志的事务可见性判断。

## undo 日志是什么

undo 日志通常是数据库系统内部用来做事务回滚和数据恢复用的。undo 日志是数据库在执行事务时记录的一种日志信息，它保存了事务对数据进行修改之前的原始值。当需要回滚事务或系统崩溃恢复时，数据库可以使用这些信息将数据恢复到事务开始前的状态。

undo 日志维护数据的多个版本，借助 undo 日志可以实现多版本并发控制（MVCC）。

### kmstore undo 日志设计

kmstore 的 undo 日志目前包含如下信息：
1. row_id: undo log 对应的数据行
2. m_type：undo 日志的类型，什么操作产生的 undo 日志
3. m_ts：undo 的事务 id，也是 cts
4. older_ts_：上一个操作日志对应的事务号，事务提交后变成 cts
5. older_undo_entry：上一个操作的undo日志，用于产生版本链

undo 日志类型有很多，本文主要介绍：
- INSERT：插入日志
- DELETE：删除日志
- SELECT_FOR_UPDATE：有锁读
- INPLACE_UPDATE：原地更新

## 事务可见性判断

事务开始时获取开始时间戳 sts，提交时获取提交时间戳 cts。sts 和 cts 并非真正的时间戳，而是标识先后顺序的 uint64 类型整数，并且只有低 63 位有效，最高位始终为 0。对于活跃事务，还未提交，所以没有 cts，但其 cts 被我们用来表示事务 id，事务 id 最高位为 1。也就是说，一个 uint64 类型的整数被分为了两个部分，最高位为状态标志，低63位才是时间戳或事务id，如下所示：
```
uint64 值 = [1位状态标志][63位时间戳或事务ID]
```

这样设计能够保证事务 id（最高位为1）永远大于 sts 和 cts，可以更方便地判断事务的可见性：如果一个事务 a 的 sts 大于另外一个事务 b 的 cts，即事务 a 在事务b 后发生，那么 a 能够看到 b 的修改，
判断的逻辑就是：`a.sts > b.cts`，如果 b 尚未提交，那么 a.sts 必然小于 b.cts，所以 a 也就无法看到 b 的修改。


事务可见性核心逻辑如下所示：

```c++
auto canSee = [ & ](uint64 next_undo_ts) -> bool {
    return (next_undo_ts == txn_cts) || (next_undo_ts < txn_sts);
};
```

这段代码用于遍历 undo 列表，找到第一个本事务可见的版本，是 kmstore MVCC 核心，含义如下：
1. 本事务的修改对本事务可见
2. 其他事务修改的版本，如果其提交时间戳（cts）小于当前事务的开始时间戳（sts），即发生在当前事务之前则可见，否则不可见。


## undo 版本链

kmstore 内部维护数据的历史版本，具体而言是 undo 链，下面我们从事务流程角度，一起看看 undo 的设计与实现。

### 事务流程

#### 开启事务

kmstore 事务的开启是在第一次 lock_tables 阶段。

开启事务会创建事务上下文，这里是 `DMLTransactionContext`，并且获取全局自增的 sts，cts 在 sts 基础上将最高位置 1。

```c++
 cts = std::make_shared<std::atomic<cid_t>>(sts | (1ULL << 63));
```

代码流程如下：
```c++
ha_external_lock -> external_lock --> 创建 DMLTransactionContext
                                  `-> 开启事务 --> 获取 bucket id
                                             |-> 添加到活跃事务列表
                                             |-> 获取 sts
                                             `-> cts：将 sts 的最高位置1
```

全局事务管理器中维护活跃事务列表和已完成的事务列表：
```c++
  std::array<std::unique_ptr<TransactionSTSWithLock>, TXN_MAP_BUCKETS> active_txn_map_array_;
  std::array<std::unique_ptr<TransactionCTSWithLock>, TXN_MAP_BUCKETS> finished_txn_map_array_;
```

活跃事务列表和已完成的事务列表都是大小固定为 64 的哈希桶，开启事务时，会根据事务的 bucket id 找到对应的桶。
桶的实现是带锁的队列，每个事务在获取 sts 时需要先对桶加锁，然后获取全局自增的 id，再将 sts push 到槽中。
也就是说，每个桶按序保存 sts。

通过分桶，实现细粒度锁操作，提升获取 sts 的性能。


#### 写入

插入数据流程如下：
```c++
ha_write_row -> write_row -> km_row_insert -> InsertRow --> InsertTuple
                                                        |-> PushInsert
                                                        `-> InsertUndoEntryToFront：将undo log存入mapping table
```

InsertTuple 负责将数据插入行存页，如果行存页空间不足，会尝试获取新的行存页。当数据插入成功以后，会为 Insert 操作生成一个 UndoLogEntry，UndoLogEntry 中保存插入的 row_id，事务id（以 cts 形式）等信息。

insert 生成的 undo 日志，会保存在两个地方：
1. 事务上下文中
2. insert mapping table 中

事务上下文是线程私有的，存储某个事务的信息。事务的操作保存在事务上下文中，很好理解。下面介绍一下 insert mapping table。

insert mapping table 保存数据与版本链的映射关系，每个 row page 对应一个 insert mapping table，当我们往 row page 中插入一行时，也会往 insert mapping table 中插入一行，insert mapping table 保存 row id 和它对应的 undo 日志链。

核心数据结构：
```c++
vector<array<UndoLogEntry*, 128>>
```

这个数据结构说明了 insert mapping table 的设计原理：insert mapping table采用了等高的桶，每个桶可以保存 128 个 undo 指针，桶的总数是不固定的（vector）。insert mapping table 的设计是保存每个 row id 的 undo 指针，row id 是连续的，所以你可以理解为第一个桶保存 `1~128` 行，第二个桶保存 `129~256`，依次类推。采用这种预分配空间的策略，访问 insert mapping table 可以不用加锁，能够提升性能。当然在 vector 扩容时，目前是需要加锁的。

insert 操作生成的 undo 日志通常是版本链的最后一个节点，那么版本链的头节点保存在哪里呢？

头结点保存在 mapping table 中，mapping table 和 insert mapping table 稍微有一些不同。

mapping table 的核心数据结构如下：

```c++
array<<unordered_map<int, UndoLogEntryHeader*>, 16>
```

mapping table 包含固定的 bucket，目前为 16，每个 bucket 实际是一个哈希表，保存 row id 和 undo 版本链的头结点。对比可以发现，insert mapping table 的桶数是不固定的，但是每个桶大小是固定的；mapping table 的桶数是固定的，但是桶的大小是不固定的。


TODO：

1. 为什么数据结构要这样设计，有什么好处？
2. 为什么要单独设计一个 insert mapping table？


#### 更新

更新数据形成版本链，update 首先会执行 select for update，然后再执行 update。

select for update 流程如下：

```c++
read_record -> km_row_select_row_page_with_mvcc --> build_prev_ver_for_consistent_read
                                                `-> km_row_select_row_page_to_kstore --> GenUndoLogEntry
                                                                                     `-> RegisterUndoLog
```

当执行 select for update 读取某一行时，首先为这一行生成一个类型为 `SELECT_FOR_UPDATE` 的 undo 日志，并将这个 undo 日志挂到版本链上，同时也会将这个 undo 保存到事务上下文中。我们注意到在读取数据之前，还存在一步 `build_prev_ver..`，这个步骤中会判断事务的可见性，在下文中会详细说明。

通过 select for update，我们往版本链中插入了一个有锁读的 undo 日志，update 更新会找到刚刚 select for update 插入的 undo 日志，然后将 undo 日志的类型由 `SELECT_FOR_UPDATE` 更新为 `INPLACE_UPDATE`。

更新数据的流程如下：
```c++
update_row -> km_row_update -> UpdateRowLocked --> GetUndoEntryHead(row_id)
                                               |-> SetType(UndoEntryCode::IN_PLACE_UPDATE);
                                               `-> SetValues(indexed_values);
```
update 调用的是 `GetUndoEntryHead` 获取版本链的第一个 undo 日志，这个 undo 日志就是 select for update 生成的。select for update 会加行锁，这样就保证了在更新操作之前，其他事务无法操作该行。

update 的 undo 日志还会记录哪些列被更新了，这些列更新之前的数据是什么，这些信息保存在 `indexed_values` 中，通过 `SetValues` 方法存入 undo 日志。

#### 删除

删除操作与更新操作比较类似，都会先执行一个 select for update 操作，插入一个 undo 日志，并在实际操作之前改变该 undo 日志的类型。update 操作将 undo 日志的类型改为 `IN_PLACE_UPDATE`，delete 操作将日志类型改为 `DELETE`。

删除数据的流程如下：

```c++
delete_row -> km_row_delete --> GetUndoEntryHead(row_id)
                            |-> SetType(UndoEntryCode::DELETE)
                            |-> rp->SetDeleted(row_id)
                            `-> SetValues(indexed_values);
```

删除操作也会在 undo 日志中记录删除之前的数据。


#### 查询

在 `build_prev_ver_for_consistent_read` 中会进行遍历版本链，找到第一个当前事务可见的版本。

如果说是有锁读，那么直接可以看到最新的数据，而不会遍历版本链。

遍历版本链的过程如下：

```c++
  cur_undo_log = undo_header.undo_log_entry;
  do {
    switch (const auto cur_undo_type = cur_undo_log->GetEntryType(); cur_undo_type) {
      case UndoEntryCode::DEL_INS: {
        ...
      }
      case UndoEntryCode::INSERT: {
        assert(!cur_undo_log->GetOlderEntry());
        return ResType::NOT_FOUND;
      }
      case UndoEntryCode::MOV_UPD: {
        ...
      }
      case UndoEntryCode::IN_PLACE_UPDATE: {
        for (const auto &i : cur_undo_log->GetValues()) {
          modifications[i.column_offset] = i.value;
        }
        break;
      }
      case UndoEntryCode::SELECT_FOR_UPDATE:
      case UndoEntryCode::SET_BIT:
      case UndoEntryCode::DELETE: {
        break;
      }
      default: {
        assert(0);
      }
    }
    cur_undo_ts = cur_undo_log->GetOlderTS()->load();
    cur_undo_log = cur_undo_log->GetOlderEntry();
  } while (!canSee(cur_undo_ts));
```

遍历版本链的过程就是一个简单的 `do..while` 循环，直到找到一个可见版本。可以看到，除了上文中提到的 undo 日志类型以外，kmstore 还有一些其他类型的 undo 日志。对于不同的 undo 日志，有不同的处理逻辑：
- 如果当前是一个 insert 日志，那么它应该就是版本链的最后一个节点，并且 insert 对当前事务不可见，直接返回 `NOT_FOUND`
- 如果当前是一个 update 日志，那么将它所做的修改记录到 `modifications` 中，如果有多个 update 日志，`modifications` 会被先发生的事务覆盖

通过遍历版本链，我们能够找出对当前事务可见的一个版本，并且能够确保可重复读。

#### 提交事务

事务提交的流程如下：
```c++
ha_commit_low -> kmemstore_commit -> TransactionCtx::Commit --> 获取 cts
                                                            |-> CommitTransaction
                                                            `-> txn_manager.GCTransaction
```

- 提交事务的第一步就是获取 cts，前面我们说过，事务未提交时，cts 表达的是事务 id，其最高位为 1，在事务提交时，获取 cts。由于 cts 是一个 shared_ptr，所以该事务所有的 undo 日志中保存的时间戳都会更新成 cts。

- `CommitTransaction` 的作用是将事务产生的所有的 undo 日志保存到事务的 `undo_list` 中，起到一个汇总的作用。

- `GCTransaction` 是将这些 undo 日志交给 GC 线程，由 GC 线程回收


#### 事务回滚

所谓的事务回滚，就是消除事务产生的所有影响，事务更新了数据，那么就需要将原来的数据重新写回去，整个过程相对于提交事务稍微复杂一点。

kmstore 事务回滚的过程，就是找出事务所有的 undo 日志，并且调用 undo 日志的 `UndoLogEntry::RollbackUndoEntry` 函数。

下面的代码是 `RollbackUndoEntry` 的核心逻辑，可以看到，undo 日志的回滚也就是反向操作的过程：
- 对于 insert 的数据，进行标记删除
- 对于 delete 的数据，去除删除标记
- 对于 update 的数据，将原来的数据重新写回 row page

```c++
void UndoLogEntry::RollbackUndoEntry() {
    auto &rows = Rows(); // undo 日志中会保存事务操作过的所有的 row id
    if (rows.empty()) {
        return;
    }

    for (auto row_id : rows) {
      retry_get_page:
        const auto frp = FrameAndPage::FromRowId(table, row_id);
        auto row_page = frp.GetRowPage();

        ScopedRowLockGuard lock_row(row_page->GetRowLock(row_id), true);
        switch (this->GetEntryType()) {
          case UndoEntryCode::INSERT: {
            // revert the append in the page
            row_page->SetDeleted(row_id);
            row_page->GetMappingTable()->RollbackInsert(row_id);
            break;
          }
          case UndoEntryCode::DELETE: {
            // restore the deleted flag on rollback
            row_page->RestoreDeleted(row_id);
            row_page->GetMappingTable()->RemoveFirstEntry(row_id);
            break;
          }
          case UndoEntryCode::SELECT_FOR_UPDATE: {
            row_page->GetMappingTable()->RemoveFirstEntry(row_id);
            break;
          }
          case UndoEntryCode::IN_PLACE_UPDATE: {
            // restore the old value in the page
            const std::vector<indexed_value> &values = this->GetValues();
            uint32_t nvlo = 0;
            for (auto &indexed_value : values) {
              row_page->SetValue(row_id, indexed_value.column_offset, nvlo, indexed_value.value, true);
            }
            row_page->GetMappingTable()->RemoveFirstEntry(row_id);
            break;
          }
        }
    }
}
```

事务回滚完成后，也需要调用 `txn_manager.GCTransaction` 将 undo 日志交给 GC 线程。

## undo 日志回收

什么样的 undo 日志可以被回收呢？简单来说：
当某个事务对所有事务都可见的时候，它的所有的 undo 日志就可以被回收。

过程如下：
1. 事务完成后，会将 undo 日志保存到完成事务列表中
2. GC 线程通过遍历活跃事务列表，找出最早开始的事务，将其 sts 记为 `global_min_sts`，
3. 遍历以完成事务列表，找到所有的 cts 小于 global_min_sts 的事务的 undo 日志
4. 释放 undo 日志内存，对于 insert undo，还需要将对应的 insert mapping table 置空


下面是具体代码细节分析：

在事务提交或者回滚的时候，会调用 `GCTransaction` 将 undo 日志交给 GC 线程。`GCTransaction` 如下所示：

```c++
  void GCTransaction(const cid_t unique_id, const cid_t sts,
                     std::vector<std::unique_ptr<kmemstore::mvcc::UndoLogEntry>> undo_log_list) const {
    active_txn_map_array_[txnIDMod(unique_id)]->remove_txn_from_queue(sts, std::move(undo_log_list));
  }
```

活跃事务列表对应的数据结构是 `TransactionSTSWithLock`，它的内部用一个队列保存 sts，如下所示：

```c++
class TransactionSTSWithLock {
  TransactionCTSWithLock &cts_min_heap;
  std::unordered_map<cid_t, std::vector<std::unique_ptr<kmemstore::mvcc::UndoLogEntry>>> deleted_txn_cache;
 // 按 sts 排序的 txn_id 队列
  std::queue<cid_t> sts_queue;
  mutable std::mutex sts_queue_mutex;
}
```

`TransactionSTSWithLock` 的 `remove_txn_from_queue` 能够保证按照 sts 的顺序将 undo 日志交给回收线程：如果一个事务并非这个分桶内最早的事务，那么它的 undo 日志
会被保存在 deleted_txn_cache 中，直到所有早于它事务都完成之后，它的 undo 日志才会交给 gc 线程。

这个行为导致”活跃事务“的含义稍微发生了一点改变：一个事务结束后还会保留在活跃事务列表中，直到与它同一个分桶的其他事务都完成。

当一个事务”真正地”结束后，我们会从 `sts_queue` 中将其移除，并将 undo 日志保存到 `TransactionCTSWithLock` 中（一个 `TransactionSTSWithLock` 对应一个 `TransactionCTSWithLock`）。

`TransactionCTSWithLock` 内部采用优先级队列维护所有已完成事务的 undo 列表，如下所示：

```c++
class TransactionCTSWithLock {
  using Item = UndoList;

 private:
  std::priority_queue<Item, std::vector<Item>, CTSComparator> cts_min_heap;
  std::mutex cts_heap_mutex;
}
```

使用 priority_queue 是为了更方便地找出小于 `global_min_sts` 的事务的 undo。


GC 线程会计算全局最小的 sts（global_min_sts），计算 global_min_sts 会遍历活跃事务列表，找出最早开始的事务，将其 sts 记为 global_min_sts，过程如下；

```c++
  for (const auto &arr : active_txn_map_array_) {
    minSts = std::min(minSts, arr->minTs());
  }
  global_min_sts = minSts;
```

拿到 global_min_sts 之后，会将遍历完成事务列表，收集可以会被回收的 undo list，启动 `FreeTxnCtxWorker` 任务进行回收处理。
```c++
    for (size_t i = 0; i < finished_txn_map_array_.size(); ++i) {
      auto &gc_txn_list = finished_txn_map_array_[i]->remove_txn_from_min_heap(global_min_sts);
      for (auto &undo_list : gc_txn_list) {
        thread_pool.enqueue(&FreeTxnCtxWorker, std::move(undo_list), global_min_sts);
      }
      gc_txn_list.clear();
    }
```

`FreeTxnCtxWorker` 中的主要逻辑是为了处理 insert undo 日志，找到它在 insert mapping table 中的位置，并将该位置的指针置为空。