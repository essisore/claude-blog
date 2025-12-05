---
layout: ../../layouts/BlogPost.astro
title: "mariadb 向量索引源码笔记"
description: "vector index in mariadb"
publishDate: "2025-12-03"
tags: ["vector database", "mysql", "hnsw"]
---

宋昭大佬已经在 [MariaDB vector index 实现分析](https://zhuanlan.zhihu.com/p/1904733400474054912) 用通俗易懂的语言描述了 mariadb 的 vector index 实现原理，以及性能分析。

通过这篇文章，我们知道了：
1. 向量索引是在 Server 层实现的，innodb 不感知
2. 向量索引对应一张内部表
3. 向量类型内部其实是 varchar

内部表：

```sql
CREATE TABLE `用户表表名 + #i# + vector列位置` (
  layer tinyint not null, -- 层数
  tref varbinary(%u), -- 用户表的主键值，用于内部表记录与对应用户表记录的关联
  vec blob not null, -- 向量数据，与主表存储方式不同
  neighbors blob not null, -- 该节点在每一层的邻居，存的是 DB_ROW_ID
  unique (tref),
  key (layer)
)
```


另外需要补充的是：
mariadb 的向量索引基于 hnsw 算法，并做了修改。关于 hnsw 可以参考我之前的博客「HNSW 索引实现（faiss）」。

在开始阅读本文之前，请先考虑几个问题：
1. hnsw 是面向纯内存设计的数据结构，用表保存 hnsw 之后，会涉及到磁盘 io。那么如何应对 hnsw 的随机读问题，mariadb 有没有相应的优化？
2. mariadb 是如何保证事务的？回滚时如何撤销对 hnsw 的修改？

本文着重于源码实现，基于 mariadb 11.8 版本，并从代码中找寻这些问题的答案。

## 数据类型

用户使用 `VECTOR` 关键字指定向量数据类型，如下所示：

```sql
VECTOR(N)
```


用法示例：

```sql
CREATE TABLE products (
    name varchar(128),
    description varchar(2000),
    embedding VECTOR(4) NOT NULL,
    VECTOR INDEX (embedding) M=6 DISTANCE=euclidean
);

INSERT INTO products (name, description, embedding)
VALUES ('Coffee Machine',
        'Built to make the best coffee you can imagine',
        VEC_FromText('[0.3, 0.5, 0.2, 0.1]'))
```

### 实现原理

对外显示 vector 数据类型，但在内部，mariadb 使用 varchar 存储数据。

mariadb 使用 udt 机制来扩展支持 vector 类型。

当往 vector 列插入数据时，需要通过 `VEC_FromText` 函数将向量数据转换为 varchar 数据：其工作原理是将向量数据当作 json 的数组进行解析，对于 json array 的每一项，使用 `strntod` 将字符串转为 `float`，再通过 `float4store` 转为4字节二进制，最终将这些二进制 append 到一起。所以，`vector(n)` 占用的空间为 `n * 4` 字节。

### 代码分析

vector 匹配的语法规则如下所示：

```c++
udt_name float_options srid_option
{
  if (Lex->set_field_type_udt(&$$, $1, $2))
    MYSQL_YYABORT;
}
```

`set_field_type_udt` 中能够识别并返回 `type_handler_vector` 实例，这是一个全局变量，类型为 `Type_handler_vector`，处理 vector 类型。

在建表流程中，会调用到 `Type_handler` 的 `make_table_field_from_def` 接口完成转换，将 Type_handler 转换为 Filed，vector 类型对应的类型为 `Field_vector`。Field 中保存了 server 层和存储层数据交互的桥梁： `uchar		*ptr;` 指针，通过 Field 的 `store` 方法，可以将数据保存在 ptr 所指向的内存中，并交给存储引擎。

Field_vector 是 Field_varstring 的子类，其 store 方法会最终调用 Field_varstring 的 store 方法。

Field_vector 只实现了接收 `const char *` 的 store 方法。在往 vector 类型插入数据前，必须要调用 `VEC_FromText` 函数将向量数据转换为二进制表示方式。

`VEC_FromText` 对应的类为：`Item_func_vec_fromtext`，其 val_str 完成将向量类型字面值转换为二进制，之后调用 Field_vector 的 store 函数存入 ptr 中。

## 向量索引

### 索引创建流程

mariadb 新增了 high level index（hlindex）机制，来支持创建向量索引。通过在建表流程（对应 `ha_create_table` 函数）中增加对 hlindex 的处理，完成创建内部表的逻辑。


#### 源码分析

hlindex 的判断逻辑：

```c++
uint hlindexes() { return total_keys - keys; }
```

- total_keys：表示创建表时指定的所有索引的数目，包含 hlindex，目前 hlindex 就是指向量索引
- keys：存储引擎上创建的索引数目

在 ha_create_table 的最后，如果检测到表上含有 hlindex，就会创建内部表，步骤如下：

```c++
ha_create_table {
    // ... 正常的建表流程
    if (share.hlindexes()) {
        1. init_tmp_table_share // 初始化临时 index_share
        2. mhnsw_hlindex_table_def // 取回建表语句
        3. index_share.init_from_sql_statement_string // 用建表语句创建来初始化 index_share
        4. ha_create_table_from_share // 创建临时表
    }
}
```

### hnsw 数据结构构建

在往表中插入数据时，会构建 hnsw 索引，整体流程如下：

![mariadb 构建 hnsw 索引流程](/assets/images/maria-build-hnsw.svg)


### 索引查询

### 事务保证分析

## 性能

## 参考资料
1. [MariaDB vector index 实现分析](https://zhuanlan.zhihu.com/p/1904733400474054912)
2. [How Fast Is MariaDB Vector?](https://mariadb.com/resources/blog/how-fast-is-mariadb-vector/)
