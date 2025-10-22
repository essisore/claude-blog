---
layout: ../../layouts/BlogPost.astro
title: "apply 算子的实现笔记"
description: "介绍数据库的 apply 算子以及如何实现"
publishDate: "2023-07-31"
tags: ["database", "kundb", "executor"]
---

## 介绍

apply 算子用于计算关联子查询，其来源于微软 SIGMOD 2001 的[论文](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.563.8492&rep=rep1&type=pdf)：

论文给出了 Apply 的定义：

$$
R \ A^\otimes \  E=\bigcup_{r \in R} (\{ r \} \, \otimes E(r) )
$$

> Apply takes a relational input R and a parameterized expression
E(r); it evaluates expression E for each row r ∈ R, and
collects the results.
where ⊗ is either cross product, left outerjoin, left semijoin, or left antijoin.
>

```
     apply
    /     \
outer    inner
  R      E(r)
```

通俗地讲，就是：

 **“遍历 outer，对于 outer 当中的每一行 r，带入到 inner 侧，计算表达式 E(r) 的结果，并根据不同 apply 类型决定如何输出数据”**

P.S. 这描述的不正是子查询的计算逻辑嘛

Apply 同样能够表示非关联子查询，此时 E(r) 为 E()，即表达式 E 的计算结果不依赖 R。

SQL Server 在 sql 语法中提供 cross apply 和 outer apply 支持：

[FROM clause plus JOIN, APPLY, PIVOT (T-SQL) - SQL Server](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql?view=sql-server-ver16#syntax)

### 子查询分类

1. 按照是否依赖外部数据分类：关联子查询、非关联子查询
2. 按输出数据的形式分类：scalar、column、row、table
3. 按照作用分类：
    1. 存在性检测：exists 子查询
    2. 量化比较：形如 column op [any/all] ( subquery )，op 可以是 >、<、= 等， in子查询
    3. scalar 子查询：任何标量表达式都可以用 scalar 子查询替代
4. 按照子查询所处的位置：select子查询、from子查询（即 derived table）、where 子查询等

举例：

```sql
-- 量化比较
select 1 from t1 where c1 > (select count(*) from t2 [where id = t1.id]);
select c1 > (select count(*) from t2 [where id = t1.id]) from t1;
select 1 from t1 where c1 > [all|any|some] (select c1 from t2 [where id = t1.id]);

-- scalar subquery
select (select c1 from t2 [where id = t1.id]) from t1;
select (select max(c1) from t2 where id = t1.id) from t1;
select (select max(c1) from t2 where id = t1.id) g from t1 having g is not null;

-- 存在性检测
select id [not] in (select id from t2) from t1;
select * from t1 where id [not] in (select id from t2);
select [not] exists (select * from t2 where id = t1.id) from t1;
select * from t1 where [not] exists (select * from t2 where id = t1.id);

-- derived table
select * from t1 join (select * from t2) as foo on t1.id = foo.id;
select * from t1
  left join LATERAL (select id from t2 group by id having max(c2) < t1.c2) foo;
```

### Apply 算子的类型

- [**anti] semi apply**

    semi apply：对于 outer 的每一行 r，将 r 代入 inner 中计算 E(r)，如果 E(r) 不为空则输出 r，否则丢弃

    anti semi apply：与 semi apply 相反，如果 E(r) 为空则输出 r，不为空则丢弃

- **cross apply**

    对于 outer 的每一行 r，计算 E(r)，并将 r 和 E(r)  join 起来输出

- **left outer apply**

    对于 outer 的每一行 r，计算 E(r)，并将 r 和 E(r)  join 起来输出，如果 E(r) 为空，则对 r 进行补空后输出


> These various forms of Apply have been introduced for convenience, and they contribute to a practical, efficient implementation, but they can all be expressed based on Cross Apply, whose purpose is to abstract parameterization.
>

## 子查询转换成 Apply 算子

> In general, Left Outer Apply will be used, so that a NULL value is passed up when the correlated subquery returns an empty value.
>

如下面这条 sql：

```sql
select * from t1 where exists (select * from t2 where id = t.id);
```

众所周知，exists 的结果是 true 和 false，当 exists 后的子查询结果集不为空，exists 返回 true，否则返回 false，然而数据库实现中通常不会实现 exists 表达式，而是通过 semi join 直接输出结果。

上述 sql 中，对于 t1 的每一行 r，如果 t2 中存在 id 与 r.id 相等，exists 返回 true，r 就能输出；反之，如果 t2 中不存在 id 与 r.id 相等，那么 r 就不能输出。

这正符合 semi apply 的定义，R 为 t1，E(r) 为 select * from t2 where id = t.id，对于 R 的每一行 r，当 E(r) 为空时，r 被丢弃；当 E(r) 不为空时，输出 r。

如果 sql 支持 semi apply 的语法，那么上述 sql 应当等价于：

`select * from t1 R semi apply (select * from t2 where id = R.id) E`

in 子查询和 exists 子查询很类似，但是其对于 null 值的处理是有所不同。分析下面这条 sql 的输出：

```sql
select * from t1 where id in (select id from t2);
```

对于 t1 的每一行 r，讨论 r.id 是否为 null

1. r.id 为 null 时，那么 in 表达式为 null，此时 where 评估为 false，故 r 不输出
2. r.id 非 null 时，假设其值为 value。要么 t2 的 id 包含 value，此时 in 表达式为 true，输出 r；要么 t2 的 id 不包含 value，且不包含 null，此时 in 表达式时为 false，丢弃 r；要么 t2 的 id 不包含 value，但是包含 null，此时 in 表达式为 null，where 整体为 false，丢弃 r

分析 select * from t1 where exists (select * from t2 where id = t.id); 我们发现两者的输出是一样的。实际上， where .. in 等价于 exists，故也可以用 semi apply 实现。

再看下面这条 sql，与上条 exists 很像，但位置不同：

```sql
select exists (select * from t2 where id = t.id) from t1;
```

这条 sql 输出的结果是 exists 表达式的计算结果，即 true 或者 false，在 MySQL 中是 1 或者 0。这条 sql 无法使用 semi apply 来表达，因为 semi apply 只能输出 r，而不能输出 true 或者 false。

对于 t1 的每一行 r，如果 t2 中存在 id 与 r.id 相等，此时不是输出 r，而是输出 true；如果不存在 id 与 r.id 相等，不是丢弃 r，而是输出 false。

in 子查询和 exists 子查询很类似，但是其对于 null 值的处理是有所不同。下面这条 sql 与上述 sql 又略有不同，它还可能输出 null，即当 id 为 null 或者 id 为某个 key，但是 t2 的 id 没有 key 且含有 null 时，id in (select id from t2) 的结果为 null。

```sql
select id in (select id from t2) from t1;
```

为了支持上述 sql，我们需要为 apply 算子引入 mark 属性。

### mark 属性

apply 算子包含一个 mark 属性，如果 mark 为 true，不仅需要输出结果集，还需要输出匹配过程。输出匹配过程是指，对于结果集的每一行还需要最后额外输出一列 bool 列，取值为 true、 false 或者 null，表示是否匹配。

目前只有 [anti] semi apply 支持 mark 属性， cross 和 left outer apply 会忽略 mark 属性。

mark 属性会改变 [anti] semi apply 算子的输出逻辑，原本不输出的 r，现在会输出。

添加 mark 属性后，Apply 算子的类型中无法用 semi apply 计算的 exists 和 in 子查询就可以使用 semi apply 来计算。

## Apply Condition

apply condition 是我们为了解决 in 子查询能够输出 null 而引入的。

通常而言， apply 算子没有匹配条件，但对于 `col1 in (select col2 ...)`，在构建 apply 算子时，会构造 `col1 = col2` 作为 apply 的条件，当 mark 为 true 时，根据 `col1 = col2` 的结果来决定输出的 bool 列是什么。

## 设计

### 角色

4个角色，3条数据 channel

- outerRowFetcher：负责从 outer executor 中不停地获取数据，将数据一行行地发送给 outerRowCh
- dispatcher：根据 outer row 更新 inner 算子，不停地消费 inner 数据，将数据一批批地发送给 matcherInputCh
- matcher：一个 matcher 负责一个 batch 的数据，将 outer row 与 batch 做匹配，将匹配结果发送给 collectorCh
- collector：负责收集 outer row 和所有 batch 的匹配结果，通过 chunk 将数据返回给 apply 算子的父节点

### 具体步骤

1. 起一个 outerFetcher 不停地获取 outerExec 的数据，并将数据以行为单位放到 outerRowCh 上
2. main thread 充当 dispatcher 角色，不停地从 outerRowCh 上获取 outer row
3. 对于每一行 outer row，起若干 matcher 和一个 collector
4. 对于每一行 outer row，dispatcher 不停地获取 innerExec 的结果集。每获取一个 batch 的数据，就将数据发送给 matcherInputCh。dispatcher 通过往 matcherInputCh 上写来派发任务。
5. matcher 从 matcherInputCh 上读取 innerExec 的一批数据，简称 batch，并用 outer row 和 batch 进行匹配。最后将（匹配的结果集，是否匹配，error) 发送给 collectCh。需定义 “匹配” 接口，用于处理不同类型的 apply。
6. collector 从 collectCh 上收集每一个 batch 匹配的结果，并做最后地输出
7. dispatcher 等待所有的 matcher 和 collector 完成工作后，方可迭代下一行

## 参考：

1. Orthogonal Optimization of Subqueries and Aggregation

    [](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.563.8492&rep=rep1&type=pdf)

2. The Complete Story of Joins (in HyPer)

    [](https://www.btw2017.informatik.uni-stuttgart.de/slidesandpapers/F1-10-37/paper_web.pdf)

3. SQL 子查询的优化

    [SQL 子查询的优化](https://ericfu.me/subquery-optimization/)

4. Parameterized Queries and Nesting Equivalences

    [](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2000-31.pdf)