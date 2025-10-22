---
layout: ../../layouts/BlogPost.astro
title: "优雅地关闭 go channel"
description: "学习 MySQL 查询处理的笔记之自然连接"
publishDate: "2023-07-19"
tags: ["database", "mysql", "query processing"]
---

原则：

> **don’t close a channel from the receiver side** and don’t close a channel if the channel has multiple concurrent senders.
>

更本质的原则：

> don’t close (or send values to) closed channels.
>
1. 仅有一个 sender 时，直接从 sender 端关闭即可。
2. N 个 sender，一个 receiver 时：the only receiver says “please stop sending more” by closing an additional signal channel
3. N 个 sender， M 个 receiver时：any one of them says “let’s end the game” by notifying a moderator to close an additional signal channel

在 Go 语言中，对于一个 channel，如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。所以，在这种情形下，所谓的优雅地关闭 channel 就是不关闭 channel，让 gc 代劳。