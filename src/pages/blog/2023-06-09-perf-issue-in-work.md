---
layout: ../../layouts/BlogPost.astro
title: "工作中的小故事之索引"
description: "那些钻牛角尖的时刻"
publishDate: "2023-06-09"
tags: ["database", "mysql", "perf"]
---

## 神奇的现象

对于上述 sql，where 条件其实永远是 true，但是当我们将 where 条件去掉以后，发现执行时间从 0.9s 飙升到 9s 。

```sql
SELECT
  count(1) AS total
FROM
  (
    SELECT
      b.user,
      b.db_name,
      a.session_id,
      a.gate_id
    FROM
      general_metrics AS a
      LEFT JOIN session_metrics AS b ON a.session_id = b.session_id
      AND a.gate_id = b.gate_id
    WHERE
      (
        '' = ''
        OR sql_id LIKE concat('%', '', '%')
        OR sql_template LIKE concat('%', '', '%')
        OR user LIKE concat('%', '', '%')
        OR db_name LIKE concat('%', '', '%')
      )
    GROUP BY
      a.sql_id
  );
```


## 为什么?

通过 explain 展示此 sql 的执行计划（已处理），如下所示：

```sql
└─ LookupJoin    | LeftJoin, Join Condition: LogicalAnd(ComparisonEqual(#1, #30), ComparisonEqual(#2, #31))
   ├─ *TableScan | {select a.session_id, a.gate_id, a.sql_id, a.sql_template from general_metrics as a}
   └─ *TableScan | {}
```

可以看到，此 sql 在 KunDB 中使用了 lookup join。lookup join 是一种 join 算法实现，会根据左表的数据构造过滤条件应用到右表上，从而减少右表在网络上的传输量，以达到性能优化的目的。在 lookup join 实现中，对于左表的每批次数据，会按照 join key，本例中是 session_id 和 gate_id， 排序去重，如果左表是按照 join key 排好序的，lookup 的次数也就越少，性能越好。

### 关键的一列

在本例中，性能差的 lookup join，即去掉 where 条件后，生成的左表查询语句是 `select a.session_id, a.gate_id, a.sql_id from general_metrics as a`，相比于性能好的少查询了一列 sql_template。**正是少查询了一列 `sql_template` 导致性能慢了 10 倍。**

本例中，对于 general_metrics 表，其主键是： "GATE_ID", "SESSION_ID", "SQL_ID", "START_TIME"，表上还存在一个 SQLID(session_id，gate_id，sql_id) 索引。

当查询 session_id，gate_id，sql_id 时，mysql 会使用 SQLID 索引，返回的数据是按照 sql_id 排序的；如果需要额外查询 sql_template 时，mysql 则不能使用 SQLID 索引，改为使用全表扫描，返回的数据是按照 gate_id, session_id 排序的。这对于 mysql 来说，选取的是最优的执行计划。

对于 KunDB 而言，**局部最优却导致了全局变差：**

当选择 SQLID 索引扫描数据时，返回给 lookup join 的数据就是乱序的。lookup join 希望按照 session_id  和 gate_id 排序，如此去重效果才会好；另外一方面，使用主键扫描，返回的数据是按照 gate_id, session_id 排序，lookup join 排序去重效果好。去重效果越好，对右表的过滤率越高，需要将右表数据取回本地的比例就越小，性能就越好。所以，根因还是减小了网络上传输的右表的数据量。