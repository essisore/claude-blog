---
layout: ../../layouts/BlogPost.astro
title: "非法的聚合函数"
description: "本文主要介绍 mysql 是如何解析聚合函数的"
publishDate: "2022-04-06"
tags: ["database", "mysql", "query processing"]
---

本文基于 mysql(8.0.28) 源代码对于 `class Item_sum` 的说明编写，纯属个人理解，如有错误，恳请指证。这段说明也可以在 mysql 的 docxygen 文档中找到：[Detailed Description](https://dev.mysql.com/doc/dev/mysql-server/latest/classItem__sum.html#details)， 如果可能，还请参阅 mysql 的官方注释。

## 整体设计

### 聚合函数的合法性

聚合函数不是在什么地方都可以用的，最简单的例子：`SELECT AVG(b) FROM t1 WHERE SUM(b) > 20 GROUP by a`。这条 sql 是不合法的，因为 sql 要求 where 条件中的表达式必须可以针对表的每一行数据进行计算，对于聚合函数的计算需放在 having 条件中。在一个最简单的 select 语句中，where 和 group 子句是不能出现聚合函数的，而像 select 和 having 子句中是允许的；但是对于包含子查询的 sql 而言，判断聚合函数的合法性并不是一件简单的事情。

是不是说 where 条件中包含的聚合函数一定是非法的呢？答案是“否”，如下所示：尽管子查询的 where 条件中出现了聚合函数：sum(t1.b)，但是它却是合法的，因为整个子查询位于外层查询的 having 子句中。

`SELECT t1.a FROM t1 GROUP BY t1.a HAVING t1.a > ALL (SELECT t2.c FROM t2 WHERE SUM(t1.b) < t2.c)`

上面这条合法的sql是怎么回事呢？我们都知道关联子查询的计算过程：对于外表的每一行，将数据代入到子查询中替换掉那些关联列，从而可以对子查询进行计算，根据子查询计算的结果，再对外层查询进行计算。这儿，对于 t1 的先按照 a 列进行分组，对于每一个分组求出 sum(t1.b)，这样便能得到一个临时结果，一行行迭代这个临时结果，将 sum(t1.b) 代入到子查询中进行计算。

### 聚合函数的计算位置

现在看一个相似的例子，如下所示：

```sql
SELECT t1.a FROM t1 GROUP BY t1.a
 HAVING t1.a > ALL(SELECT t2.c FROM t2 GROUP BY t2.c
                     HAVING SUM(t1.a) < t2.c)
```

与上个例子不同，在这个例子中，sum(t1.a) 位于子查询的 having 子句中，因此 sum(t1.a) 既可以在外层查询中计算，也可以放到子查询中进行计算。具体而言：

- 外层：外层先按照 t1.a 进行分组，并计算 sum(t1.a)，接着将计算出来的 sum(t1.a) 代入子查询中。按照 t1.a 分组，sum(t1.a) 的结果就是 t1.a 的值乘以该值出现的次数。
- 子查询：外层按照 t1.a 进行分组，但不计算 sum(t1.a)，而是将 t1.a 的数据当作「常量」代入到子查询中，子查询按照 t2.c 分组后，再求出 「sum(常量)」 。代入常量的子查询就变成了 `SELECT t2.c FROM t2 GROUP BY t2.c HAVING SUM(1) < t2.c`，该 sql 是一个合法的 sql。「sum(常量)」的含义就是常量乘以该分组的基数。

对于分组列或者常量求聚合函数仍不理解的，可以看看下面这个简单的例子：

```sql
mysql> create table test(a int, b int);
Query OK, 0 rows affected (0.04 sec)

mysql> insert into test values(1,1),(1,2),(1,3),(2,1),(2,2),(3,1);
Query OK, 6 rows affected (0.01 sec)

mysql> select sum(a),sum(1),sum(2) from test group by a;
+--------+--------+--------+
| sum(a) | sum(1) | sum(2) |
+--------+--------+--------+
|      3 |      3 |      6 |
|      4 |      2 |      4 |
|      3 |      1 |      2 |
+--------+--------+--------+
3 rows in set (0.01 sec)
```

**在不同的位置计算聚合函数将会导致不同的结果**。

对于mysql而言，其有**一套规则**来决定聚合函数的计算位置：

> 给出聚合函数 S(E)，其中 E 是一个包含多个列引用 C1～Cn 的表达式。通过名称解析能够解析出每个列引用 Ci 对应的 query block：Qi。令 Q 为所有 Qi 中最内层的那个，如果 S(E) 在 Q 中所处的子句被允许出现聚合函数，如 having clause，那么就在 Q 中计算该聚合函数。如果所处的子句不允许出现聚合函数，如 where clause，那么：
>
> - 如果启用了 ANSI SQL 模式，则报错
> - 否则，找到最里层的能够计算 S(E) 的 query block。

这儿：S(E) 的计算位置绝不可能是包含 Q 的 query block。道理很简单，因为E中有来自 Q 的列引用，而外层查询不能引用子查询中的列。

看几个具体的例子：

1. sum(t1.a+t2.b) 包含 t1.a 和 t2.b 两个列，其中 t1.a 来自最外层的查询，t2.b 来自最外层查询的子查询。而此 sum 位于 t2 子查询的 having 子句中，所以便可以在 t2 所在的 query block 中计算 sum(t1.a+t2.b)，并将结果代入的 t3 所在的query block 中。

    ```sql
    SELECT t1.a FROM t1 GROUP BY t1.a HAVING t1.a >
        ALL(SELECT t2.b FROM t2 GROUP BY t2.b HAVING t2.b >
            ALL(SELECT t3.c FROM t3 GROUP BY t3.c HAVING SUM(t1.a+t2.b) < t3.c)
        )
    ```

2. 跟上个 case 不同，sum 位于 t2 的 where 子句中，因此我们不能在 t2 中计算该 sum。如果是 ansi sql 模式，应当报错；否则，我们就在 t3 所处的 query block 中计算该 sum。

    ```sql
    SELECT t1.a FROM t1 GROUP BY t1.a HAVING t1.a >
        ALL(SELECT t2.b FROM t2 WHERE t2.b >
            ALL (SELECT t3.c FROM t3 GROUP BY t3.c HAVING SUM(t1.a+t2.b) < t3.c)
        )
    ```

3. 该 sql 不合法，因为我们既不能在 t2 中，也不能在 t3 中计算 sum(t1.a+t2.b)

    ```sql
    SELECT t1.a FROM t1 GROUP BY t1.a HAVING t1.a >
        ALL(SELECT t2.b FROM t2 WHERE t2.b >
            ALL (SELECT t3.c FROM t3 WHERE SUM(t1.a+t2.b) < t3.c)
        )
    ```


### 嵌套聚合函数

通常情况下，聚合函数不能嵌套，如：`SELECT t1.a from t1 GROUP BY t1.a HAVING AVG(SUM(t1.b)) > 20`，尽管 avg 处于 having 子句中。

但是是不是就是说出现了嵌套的聚合函数就一定是非法的呢？答案是「否」。当出现子查询时，是允许在外层算好聚合函数并代入到子查询当中的，如下所示，对于外层查询，按照 t1.a 对 t1 进行分组，并计算每个分组的 sum(t1.b) -> s，然后将 s 代入到 t2 中，便能得到简化的查询：`SELECT t2.c FROM t2 GROUP BY t2.c HAVING AVG(t2.c+s) > 20`。显然，这条子查询是合法的。

```sql
SELECT t1.a FROM t1 GROUP BY t1.a HAVING t1.a IN
    (SELECT t2.c FROM t2 GROUP BY t2.c HAVING AVG(t2.c+SUM(t1.b)) > 20)
```

## 代码实现

本节介绍 mysql 判断聚合函数合法性的实现细节，可以帮助您更好的理解 mysql 的源代码。

- base_query_block： 聚合函数出现的 query block
- aggr_query_block： 聚合函数应当被计算的 query block
- max_aggr_level：记录聚合函数中涉及到的「未绑定列引用」所在的最深的 query block（也就是前文中提到的 Q ）的嵌套层次。嵌套 level 越大，query block 越深。啥是未绑定列引用（unbound column references）？
    - 如果聚合函数中的列引用没被绑定到任何该聚合函数中的子查询上，那么该列引用就是「未绑定列引用」

        > A column reference is unbound within a set function if it is not bound by any subquery used as a subexpression in this function.
        >
    - 而「绑定的列引用」是指那些参与到聚合函数中的子查询中某些其他聚合函数计算的那些列引用

        > A column reference is bound by a subquery if it is a reference to the column by which the aggregation of some set function that is used in the subquery is calculated.
        >

    也就是说，「绑定的列引用」得满足以下条件：

    1. 首先聚合函数的某个子表达式是子查询
    2. 子查询中包含聚合函数
    3. 参与了子查询中聚合函数的计算

    下面看两个具体的例子：

    1. 对于如下sql：

        ```sql
        SELECT t1.a FROM t1 GROUP BY t1.a HAVING t1.a >
            ALL(SELECT t2.b FROM t2 GROUP BY t2.b HAVING t2.b >
                ALL(SELECT t3.c FROM t3 GROUP BY t3.c HAVING SUM(t1.a+t2.b) < t3.c)
            )
        ```

        t1.a 是未绑定列引用，其属于主查询（ level 为 0），t2.b 是未绑定列引用，来自第一个子查询（level为 1 ），因此max_aggr_level 的值为1。很明显，**聚合函数不能在 nest_level 小于 max_aggr_level 的子查询上计算**。

    2. 对于如下sql：

        `SELECT t1.a FROM t1 HAVING AVG(t1.a+(SELECT MIN(t2.c) FROM t2)) > 0;`

        avg的max_aggr_level为0。t1.a是未绑定列引用，其对应的level为0，而t2.c是绑定的列引用，所以是不纳入考虑范围的。**如果聚合函数不包含任何列引用，如count(*)，那么它的max_aggr_level为-1。**

- max_sum_func_level：如果一个聚合函数作为子表达式成为了另外一个聚合函数的参数，并且它又并不是在外层聚合函数中的某个子查询中进行计算的，那么 max_sum_func_level 则记录着此聚合函数的最大嵌套层次。这个变量的含义是记录聚合函数子聚合函数的计算 level 的最大值。假定有两个聚合函数 s0 和 s1，s1 是 s0 的参数中的子表达式，只有当 s1 并非是在 s0 中的某个子查询中计算得到的，才可以将 s1 和 s0 看作是嵌套的聚合函数。如 `AVG(SUM(t1.b))` 是嵌套聚合，而 `AVG(t1.a+(SELECT MIN(t2.c) FROM t2))` 则不是嵌套聚合。上文中提到，嵌套聚合并不一定都是非法的。只有满足 `s1.max_sum_func_level < s0.max_sum_func_level`，该嵌套聚合才是合法的。
- in_sum_func：in_sum_func：指向包含该聚合函数的外层聚合函数。
- inner_sum_func_list：inner_sum_func_list记录着需要在本query block中额外计算的聚合函数。
- 检查聚合函数的合法性检查聚合函数合法性是在遍历其子表达式的过程中完成的。当我们向下递归遍历子表达式的时候，首先会调用 `init_sum_func_check` 来完成一些变量的初始化，在向上回朔的时候，调用 `check_sum_func` 去验证聚合函数的使用是否合法。check_sum_func 还会将该聚合函数的参数表达式链接起来。
- mysql的子查询的嵌套深度不超过63

### 深入理解 mysql 的源码实现

本部分内容是对 mysql 源码具体实现中其他一些变量的记录。

- nest_level：嵌套层次每个 query block 都有一个嵌套层次，最外层的 main query 的 nest_level 为 0，main query 的 subquery 的 nest_level 为 1。依次类推，每深入一层子查询，便增加 1。在 contextualize 阶段便构建好了，代码位于 "sql_lex.cc" 中：
    - query block 在 include_down 的时候会将自己的 nest level 加一
    - query block 在 include_neighbour 的时候，会将自己的 nest level 置为跟前面的 query block 相同
    - query block 在 include_standalone 的时候，会将自己的 nest level 置为跟「老大」相同
    - query block 在 renumber 的时候，主要是 query expression 调用，作用是更正自己的 nest level
- allow_sum_func: 当前子句是否允许出现聚合函数allow_sum_func 实际上是一个 uint64 类型的变量，但是被用作了 bitmap。它的每一位代表那一层的 query block 是否允许出现聚合函数。举个例子，比如 allow_sum_func 为 7，那就是 0x1011，最高位的 1 代表 nest_level 为 3 的 query block 允许聚合，次高位的 0 表示 nest_level 为 2 的 query block 是不允许聚合的，依次类推，最低位的 1 代表 main query block 是允许聚合的。举个例子：

    ```sql
    SELECT SUM(t1.b) FROM t1 GROUP BY t1.a
        HAVING t1.a IN (SELECT t2.c FROM t2 WHERE AVG(t1.b) > 20) AND
            t1.a > (SELECT MIN(t2.d) FROM t2);
    ```

    对上面这个sql而言：

    - 当解析到 SUM(t1.b) 的时候，allow_sum_func 的值为 0x1，因为 SELECT list中 是允许出现聚合的
    - 当解析到 AVG(t1.b) 的时候，allow_sum_func 的值也是 0x1， 这是因为 avg 处于 main query block 的 having 子句中，处于 nest_level 为 1 的子查询的 where 子句中
    - 当解析到 MIN(t2.d) 的时候，allow_sum_func 的值为 0x11，因为其处于 main query block 的 having 子句中，也处于 nest_level 为 1 的子查询的 SELECT list 中

    主要代码位于 "sql_resolver.cc"，从代码里面可以看到：

    - setup_fields 允许聚合
    - join 条件，where 子句，group by 子句中不允许聚合
    - having 和 order by 中允许聚合

    需要注意的是使用位操作设置当前 query block 的对应位，置 1 或者置 0

    - 置 1：`allow_sum_func |= (nesting_map)1 << nest_level`
    - 置 0：`allow_sum_func &= ~((nesting_map)1 << nest_level)`
- in_sum_func：每个聚合函数都有一个 in_sum_func，如果其不为空，则说明该聚合函数处于另一个聚合函数的参数中。LEX 结构体中也维护一个同名的变量，维护当前的聚合函数。当遍历表达式的时候，利用 LEX 中的 in_sum_func，就可以为每个聚合函数设置好 in_sum_func.
- max_aggr_level：这个变量的含义在前文中已经解释过了。主要的维护代码位于 "item.cc"，是在对于 Item_field 和 Item_ref 进行 fix_fields 的过程中进行赋值的，计算过程如下所述：
    - 最 normal 的case，普通列引用：列引用 a 处于聚合函数中，并且它的 query block 和 agg 本身属于同一层 query block，那么就会对聚合函数的 max_aggr_level 进行更新（取原有值和列列引用所在 query block 的 nest_level 中较大者）。e.g: `select agg(a) from t1 group by id`
    - 关联列的情况处于子查询的聚合函数中，并且聚合函数 s 的参数中的 field C1，属于外层查询 sel，并且 s 所处 query block 的 nest_level 大于等于 sel 的 nest_level，将聚合函数的 max_aggr_level 更新为原有值和 sel 中的较大者。e.g: `select a from t1 where id > (select max(t1.a) from t2)`
    - 关联ref的情况处于子查询的聚合函数 s 中，并且 R1（case中 t1.a ） 是关联 ref，并且 s 所在的 nest_level 大于等于 R1 所解析的 nest_level，则对聚合函数的 max_aggr_level 进行更新。e.g: `select a from t1 having max((select id from t2 having t1.a<100)) < 5`
- max_sum_func_level：记录计算子表达式中的聚合函数的最大值这个变量的含义在前文中讲了。在 mysql 源码中，此变量的维护和使用都是在 check_sum_func 函数当中，处理的是嵌套聚合的场景。
    1. 外层的聚合函数出现的 level 大于等于计算我的 level更新外层聚合函数的 max_sum_func_level：取原有值和计算我的 level 中较大值。通俗点讲，就是说一个聚合函数 s1 的子表达式是另外一个聚合函数 s2，计算 s2 的 query block 是 s1 出现的 query block 的外层，亦即 s1 出现的位置比 s2 计算的位置要深，这个时候就会更新 s1 的 max_sum_func_level
    2. 如果子聚合函数的 max_sum_func_level 大于我自己的则需要把我自己的 max_sum_func_level 更新为子聚合函数的
- base_query_block：聚合函数所在的 query block。聚合函数的成员变量，在 init_sum_func_check 的时候，将当前 query block 赋值给 base_query_block。
- aggr_query_block：聚合函数被计算的 query block。同为聚合函数的成员变量，在 check_sum_func 函数中，根据前文中介绍的规则，将其计算出来。