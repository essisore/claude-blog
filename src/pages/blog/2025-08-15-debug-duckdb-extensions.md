---
layout: ../../layouts/BlogPost.astro
title: "编译，调试 duckdb 社区 extension"
description: "编译调试 duckdb extension 笔记"
publishDate: "2025-08-15"
tags: ["开发环境", "duckdb", "c++"]
---

最近有需求调试 duckdb 的 [duckdb-faiss-ext](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext) 插件，简单记录一下 CLion 环境的配置。

## 配置 CLion

参考 [Setting up CLion](https://github.com/duckdb/extension-template?tab=readme-ov-file#setting-up-clion)，方法应当对所有的社区插件都是适用的。步骤如下：
1. 打开项目的时候要选 duckdb 目录
2. 然后使用 `Change Project Root`，将插件目录设置为根目录
3. 配置 CMake，将 Build directory 设置为 `../build/debug`
4. 调试时，选择 `unit_test` 作为目标，参数配置为 `--test-dir ../../.. [测试文件]`，如：
`--test-dir ../../.. test/sql/faiss6.test`

