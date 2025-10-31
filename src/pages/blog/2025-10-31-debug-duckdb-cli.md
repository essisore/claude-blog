---
layout: ../../layouts/BlogPost.astro
title: "用 CLion 调试 DuckDB CLI"
description: "Debug duckdb using clion"
publishDate: "2025-10-31"
tags: ["duckdb", "clion", "tty"]
---


我们从官网下载的 `duckdb` 可执行文件，对应的 CMake Target 为 shell，对应的 main 函数位于 `tools/shell/shell.cpp`
中。

在 `shell/CMakeLists.txt` 通过如下代码：

```cmake
set_target_properties(shell PROPERTIES OUTPUT_NAME duckdb)
```

将 shell 改名为 duckdb。

在 CLion 调试 shell 时会卡住，原因是 CLion 的 Debug Console 并非终端，而 shell 所依赖的 `linenoise` 库只能在终端下才能正常运行，所以需要开启 Debug Console 模拟终端功能。

具体请参考： [Terminal in the output console](https://www.jetbrains.com/help/clion/terminal-in-the-output-console.html)
