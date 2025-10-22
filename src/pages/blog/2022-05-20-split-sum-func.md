---
layout: ../../layouts/BlogPost.astro
title: "MySQL 查询解析笔记之拆分聚合函数"
description: "学习 MySQL 查询处理的笔记"
publishDate: "2022-05-20"
tags: ["database", "mysql", "query processing"]
---

mysql 在解析 sql 的时候会往 fields（可以理解为 select 子句中的表达式） 当中添加一些 hidden item，这些隐藏的表达式并不会输出给客户端。

本文探究在查询**解析**阶段 mysql 会往 fields 中添加哪些内容。

## 数据结构

Query_block 类中定义了 fields 和 base_ref_items 来维护 select item list，如下所示：

```cpp
mem_root_deque<Item *> fields;
Ref_item_array base_ref_items;
```

fields 是一个 deque，而 base_ref_items 是数组。下面这段注释解释了为啥 fields 选用 deque：

> This should ideally be changed into `Mem_root_array<Item *>`, but find_order_in_list() depends on pointer stability (it stores a pointer to an element in referenced_by[]).
>

其中 Ref_item_array 其实是一个类型别名，其真实定义如下：`typedef Bounds_checked_array<Item *> Ref_item_array;` ，本质上就是一个数组。

注：Bounds_checked_array 其实是一个辅助性的数据结构，类似于 c++20 中的 std::span，用以解决数组退化和越界访问。

fields 和 base_ref_items 其实是不同的数据结构描述同一项内容，在代码开发时，需要保证两者的一致性。

## 初始化

select item list 对应的 ast 数据结构是 `PT_select_item_list`。fields 在 PT_select_item_list 的 contextualize 函数中被赋值为 PT_select_item_list::value，而 value 正是 select 子句中的表达式，代码如下：

```cpp
bool PT_select_item_list::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  pc->select->fields = value;
  return false;
}
```

另外一方面，base_ref_items 数组的空间申请是在 setup_base_ref_items 函数中，通过预估将要加入到 base_ref_items 中元素的个数从而申请足够的空间。通过代码也可以看出，base_ref_items 的最终大小与 group by、order by、select cluase、having clause、聚合函数、窗口函数以及标量子查询的数量有关，代码如下：

```cpp
  uint n_elems = n_sum_items + n_child_sum_items + fields.size() +
                 select_n_having_items + select_n_where_fields +
                 order_group_num + n_scalar_subqueries;
```

base_ref_items 的初始化是在 setup_fields 的过程中完成的。在 setup_fields 时会遍历 fields，并将处理好的 item 放入 base_ref_items 中，大致代码如下所示：

```cpp
for (auto it = fields->begin(); it != fields->end(); ++it) {
	item->fix_fields(thd, item_pos);
	if (!ref.is_null()) {
      ref[0] = item;
      ref.pop_front();
  }
}
```

ref 是 base_ref_items 的临时变量，pop_front 仅仅是将 ref 向前移动了一位，因此可以通过将 item 放在 ref 的首位。

## split_sum_func

split_sum_func 和 split_sum_func2 都是用来提取表达式中的聚合函数及其参数，放入 fields 当中。

split_sum_func 是 Item 的一个虚函数，split_sum_func2 是 Item 的一个普通函数，是前者的辅助函数。在 split_sum_func2 函数的注释中详细解释了两者的用途，并举了一个很好理解的例子。

split_sum_func2 的 3 条路径：

1. 跳过那些 registered 聚合函数

    所谓 registered 聚合函数就是指子查询中的关联聚合函数。 在某些 sql 中，需要在外层计算的聚合函数被称为关联聚合函数。如果调用 split_sum_func2 时正处于子查询的处理逻辑中，那么是不应该拆分关联聚合函数的，所以直接跳过。当然，在处理主查询时不能跳过。

2. 对于满足以下4个条件之一的，调用 Item 的 split_sum_func 接管处理逻辑，不将其本身放入 fields 中
    - 本身不是聚合函数，也不是窗口函数，但是却包含聚合函数（某个子表达式是聚合函数）
    - 不是窗口函数，却包含窗口函数
    - 空值检查函数，trig_cond 函数
    - row 函数
3. 对于同时满足以下3个条件的，会将其放入 fields 当中，并用 Item_ref 替换它
    - 聚合函数 或者 包含了列引用
    - 不是子查询 或者 是标量子查询换句话说，如果是子查询，那只能是标量子查询才满足条件
    - 不是 Item_ref 或者 是VIEW_REF同理，如果是 Item_ref，那只能 ref 一个 view 才行

哪些场景会触发 split ？ P.S. 我们稍微修改一下 mysql 源码，将完成 sql 解析之后的 select item list 打印出来（上面这条是原始sql，下面的是 mysql 解析完的样子）。

1. having 条件中有聚合函数时

    例如：

    - having 子句中的 avg(c1) 会以隐藏列的形式添加到 fields当中。

        ```sql
        select id from t group by id having avg(c1)<20
         -- fields为
         	"avg(`test`.`t`.`c1`),`test`.`t`.`id` AS `id`"
        ```

        添加了 c1。

    - 另外一个 case，走的是 split_sum_func2 的第二条路径，所以 `+` 表达式并没有添加到 fields 中：

        ```sql
        select id from t group by id having id+avg(c1)<20;
         -- fields为
         	"avg(`test`.`t`.`c1`),`test`.`t`.`id` AS `id`"
        ```

        添加了 avg(c1)。

2. 主查询中包含来自子查询的聚合函数时（关联聚合函数）（此时不跳过 registered agg）

    因为此时处于 main query 的处理逻辑当中，所以不能跳过关联 agg。如：

    ```sql
    select sum(c1) from t group by id having id > (select max(c2) from t2 group by id having max(avg(t.id)+c2)>10)
    -- fields为
    avg(`test`.`t`.`id`),
    `test`.`t`.`id` AS `id`,
    sum(`test`.`t`.`c1`) AS `sum(c1)`
    ```

    可以看到在 main query 中会添加上 avg(t.id)。

3. select 子句中的表达式 **包含 wf** 或者 **不是聚合函数却包含聚合函数** 时

    例如：

    - 添加了 max(id)

        ```sql
        select max(id)+1 from t group by c1;
         -- fields为
        `test`.`t`.`c1` AS `c1`,
        max(`test`.`t`.`id`),
        (max(`test`.`t`.`id`) + 1) AS `max(id)+1`
        ```

        添加了 c1，max(id)。

    - 聚合函数里面的内容没有拆

        ```sql
        select max(id+1) from t group by c1;
         -- fields为
         	"`test`.`t`.`c1` AS `c1`,max((`test`.`t`.`id` + 1)) AS `max(id+1)`"
        ```

        添加了 c1。

    - 窗口函数的参数

        ```sql
        select c1+first_value(c2*sum(c3/c4)) over() from t;
         -- fields为
        `test`.`t`.`c2` AS `c2`,
        `test`.`t`.`c1` AS `c1`,sum((`test`.`t`.`c3` / `test`.`t`.`c4`)),
        `test`.`t`.`c2` AS `c2`,
        first_value((`test`.`t`.`c2` * sum((`test`.`t`.`c3` / `test`.`t`.`c4`)))) OVER (),
        `test`.`t`.`c1` AS `c1`, (`test`.`t`.`c1` + first_value((`test`.`t`.`c2` * sum((`test`.`t`.`c3` / `test`.`t`.`c4`)))) OVER () ) AS `c1+first_value(c2*sum(c3/c4)) over()`
        ```

        窗口函数会继续 split 其参数，最终添加到 fields 当中的有 c1，窗口函数整体，c2，sum 整体，（最前面的 c2 和 c1 并非 split_sum_func 添加的）

4. order by 子句中的表达式 **不是 agg 却包含 agg** 或者 **不是 wf 却包含 wf** 时
5. wf 中的 order by 语句中的表达式 **不是 agg 却包含 agg** 时

## order 和 group

在解析 order by 和 group by 子句的时候，如果表达式不在 select item list 中，那么就会将表达式隐藏起来加入 fields 中。

- 表达式作为整体加入到 fields 中，

```sql
select avg(id) from t group by c4, (c1+1) order by abs(c4-100);
-- fields为
	"abs((`test`.`t`.`c4` - 100)),(`test`.`t`.`c1` + 1),`test`.`t`.`c4` AS `c4`,avg(`test`.`t`.`id`) AS `avg(id)`"
```

## 总结

本文主要是介绍了 query 解析阶段，mysql 是如何构造 query block 中的 fields 的。

最主要的是处理 order by、group by以及 WF 中的 partition 和 order 子句中的表达式，将不在 select item list 中的表达式加入到 fields 中，这些子句中的表达式在 mysql 中都是通过 ORDER 这个数据结构表示，最终也都是调用 `find_order_in_list` 这个函数处理的，所以它们的处理逻辑都是相同的。

另外一大块内容就是 mysql 对于聚合函数的处理，主要是由 split 兄弟两完成的，主要功能就是将聚合函数以及其参数拆分出来加入到 fields中。

### 展望

虽然知道了 mysql 会往 fields 中添加这些内容，但是还没有完全弄明白为什么要往 fields 中添加这些内容，随着接下来对 mysql 查询处理部分的研究，应该会有一些答案。