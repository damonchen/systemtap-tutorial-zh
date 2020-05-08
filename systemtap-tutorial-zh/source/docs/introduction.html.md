---

layout: "docs"
page_title: "Introduction"
sidebar_current: "systemtap"
description: |-
关于 systemtap 的介绍
---

# 1 介绍

`systemtap` 是一个允许开发和管理员通过编写和重用简单脚本来深入检测 `Linux` 系统运行时行为的工具。可以快速且安全地提取、过滤和统计数据，用来检测复杂的性能或功能问题

**注意** ：入门手册不会描述 `systemtap` 中的每个特性，想要了解更多 `systemtap` 最新的信息，需要查看 `stap` 的手册。手册可以从安装的系统中获取，或到站点`http://sourceware.org/systemtap/man/`查看。

`systemtap`脚本的核心观点是 _events_ 和处理事件的 _handlers_。当一个特殊的事件发生时，`Linux`内核将`handler`作为子过程快速运行，然后恢复`Kernel`执行。`systemtap`定义了很多事件，如：`entering`和`exiting`一个函数，`timer`超时或整个`systemtap`会话启动与停止。`handler`是由一系列脚本语句组成的，然后在事件发生时可以做一些预定的事情，如：从事件上下文中提取数据，保存到内部变量中，打印结果。

`Systemtap` 通过将脚本转换为 `C` 语言，然后使用系统 `C` 编译器将转换后的 `C` 编译为内核模块，然后运行。当模块被加载后，在内核中就会激活所有通过钩子探测的事件。当任何一个处理器
上的事件发生时，编译后的 `handler` 就会运行，最后，当 `session` 停止时，`hook` 就断开了，然后模块就被移走了。整个过程由一个简单的命令行程序 `stap` 驱动。

```stap
# hello-world.stp
probe begin
{
  print ( "hello world\n" )
  exit ( )
}
```

```bash
# stap hello-world.stp
hello world
```

本文假设你已经安装了 `systemtap` 及内核开发工具和调试数据，此时你才能够用 `root` 账户或者 `stapdev` 中的账户或者 `sudo` 来执行上面的脚本。
