---

layout: "docs"
page_title: "analysis"
sidebar_current: "systemtap"
description: |-
本文描述systemtap的结构、语法
---

# 3 分析

一般跟踪文本的页面可能会给出足够的信息来 eexplore 系统，使用 systemstap，可以分析数据，过滤数据，aggreate 数据，转换数据并统计数据。不同的探针可以共享数据后处理。探针处理使用类似 awk 的语法形式，用一系列控制语句描述算法。使用这些方式，systemstap 脚本可以关注特殊问题，并提供压缩后的响应：无需`grep`介入。

## 3.1 基本构造

大部分 systemstap 脚本 包含条件，用来限制跟踪、处理过程或用户的一些逻辑或关注的地方，语法形式很简单：

```
if (EXPR) STATEMENT [else STATEMENT] if/else statement
while (EXPR) STATEMENT while loop
for (A; B; C) STATEMENT for loop
```

类似 C 语言，脚本使用`break/continue`，类似 awk，探针 handler 使用`next`提前返回。块语句包含在`{}`中，在 systemtap，分号作为空语句存在，不是语句结束符，所以仅`rarely`必须。shell 形式(#)，c 形式(/\*\*/)以及 c++形式(//)的注释都是可以的。

表达式类似 C 或者 awk，支持场馆的操作符，优先级以及数字量，字符串被处理为原子值，而非字符的数组，使用点来进行字符串的连接`("a" . "b")`，一些示例：

```
(uid() > 100) probably an ordinary user
(execname() == "sed") current process is sed
(cpu() == 0 && gettimeofday_s() > 1140498000) after Feb. 21, 2006, on CPU 0
"hello" . " " . "world" a string in three easy pieces
```

也支持变量，使用一个名称，然后赋值就行，然后在表达式中使用。通常是自动初始化和声明。每个标识符有类型 - 字符串或数值 - 由 systemtap 通过操作符和字面量使用方式自动推导。任何不一致处均会报告错误，字符串和数值类型的转换通过显式函数调用实现。

```
foo = gettimeofday_s() foo is a number
bar = "/usr/bin/" . execname() bar is a string
c++ c is a number
s = sprint(2345) s becomes the string ”2345”
```

默认情况，变量在探针使用中是局部的，即在每个探针的 handler 调用时初始化，使用以及销毁。要在探针中共享变量，可以脚本的在任意地方申明为全局变量。由于可能存在并发（多个探针 handler 在不同的 CPU 上运行），当 handler 运行时，每个探针使用的全局变量会自动添加读写锁。

## 3.2 目标变量

一些特殊的"目标变量"可以访问探针上下文，在符号调试器中，当在一个断点上停顿时，可以打印程序的上下文中的值。在 systemtap 脚本中，对哪些匹配到特殊执行点（除类似 timer 那样的异步事件）的探针处，也可以做类似的事情。

此外，可以获取地址（通过&操作符），pretty 打印结构（使用$和$\$后缀），pretty 打印多个变量（使用\$\$var 及相关变量），或强制转换指针为某个类型（@cast），或测试已经存在/解决的(@defined)，在手册中可以找到更多。

知道那个变量会被使用，需要在探针的时候对内核代码比较熟悉，此外，需要检查编译器不会优化不可读不存在的值，可以使用`stap -L PROBEPOINT`列出可用的变量。

假设你要跟踪文件系统对特殊`device/inode`的`read/write`操作。从内核知识中知道，需要知道两个函数`vfs_read`和`vfs_write`。每个函数需要一个`struct file*`的参数，在这个参数中，有`struct dentry *`或含`struct dentry *`的`struct path *`。`struct dentry *`包含了`struct inode*`，等等。systemtap 支持有限的指针链中的解引用。两个函数(user_string 和 kernel_string)能够将`char *`目标变量复制到 systemtap 的字符串中。

```stap
probe kernel.function ("vfs_write"),
kernel.function ("vfs_read")
{
    if (@defined($file->f_path->dentry)) {
        dev_nr = $file->f_path->dentry->d_inode->i_sb->s_dev
        inode_nr = $file->f_path->dentry->d_inode->i_ino
    } else {
        dev_nr = $file->f_dentry->d_inode->i_sb->s_dev
        inode_nr = $file->f_dentry->d_inode->i_ino
    }
    if (dev_nr == ($1 << 20 | $2) # major/minor device
        && inode_nr == $3)
            printf ("%s(%d) %s 0x%x/%u\n",
            execname(), pid(), ppfunc(), dev_nr, inode_nr)
}
```

## 3.3 函数

重复复杂的条件表达式或者日志指令在脚本中到处都是是一件很难受的事情，基于此，因此函数是软件中重用的实践方式。因此 systemtap 允许定义自己的函数。类似全局变量，systemtap 函数可以定义在脚本的任意地方。支持任意数量字符串或数值的参数（值传递），可能返回一个字符串或者一个数值。参数类型会从普通变量中推导，并需要在整个程序中保持变量类型。局部和全局脚本变量均可访问，但是目标变量不允许，这是由于没有调试级别的上下文可以和函数关联起来。

函数痛殴关键字`function`及跟随的名字来定义，然后是逗号分隔的参数列表（即变量名列表），`{}`封装部分包含了任意语句列表及调用函数的表达式，允许递归调用，但存在递归深度限制(译者注：在 4998 次以内）。

```
# Red Hat convention; see /etc/login.defs UID_MIN
function system_uid_p (u) { return u < 500 }
# kernel device number assembly macro
function makedev (major,minor) { return major << 20 | minor }
function trace_common ()
{
    printf("%d %s(%d)", gettimeofday_s(), execname(), pid())
    # no return value necessary
}
function fibonacci (i)
{
    if (i < 1) return 0
    else if (i < 2) return 1
    else return fibonacci(i-1) + fibonacci(i-2)
}
```

## 3.4 数组

通常，探针需要共享非单个简单的 scalar 值，很自然的有很多数据管道，通过线程号、进程号、名称、时间等等索引的 tuple。Systemtap 提供了关联数组来解决这个问题（译者注，和 PHP 的数组及 Lua 的 table 类似），这些数组用 hash 表实现，并需要开始时固定一个最大值。由于对于独立的探针处理运行，创建是一个很耗时的事情，所以必须定义为 global 变量

```
global a declare global scalar or array variable
global b[400] declare array, reserving space for up to 400 tuples
```

数组的基本操作是设置和查看元素，使用 awk 语法形式：数组名称，以[开始，逗号分隔的索引表达式以及]闭合，索引表达式可以是字符串或者数值，和之前脚本中的描述的类型保持一致。

```
foo [4,"hello"] ++ increment the named array slot
processusage [uid(),execname()] ++ update a statistic
times [tid()] = get_cycles() set a timestamp reference point
delta = get_cycles() - times [tid()] compute a timestamp delta
```

可以获取未设置的数组元素值，返回一空值（零或者空字符串）。但给一个元素赋值 null 并不会删除改元素，显式地`delete`语句是必须的。systemtap 提供了一个语法糖用来显式测试和删除元素。

```
if ([4,"hello"] in foo) { } membership test
delete times[tid()] deletion of a single element
delete times deletion of all elements
```

最后一个重要的是数组的迭代，使用关键字`foreach`，类似 awk，创建一个循环，对数组的 key 及 vlaue 进行迭代，迭代可能以单个 key 或者通过额外的+或-处理的值来排序

支持在 foreach 中添加 break 和 continue 语句，由于数组可以很大，但是探针 handler 必须不能长时间执行，一个比较好的方式在可能的迭代早期时退出。foreach 中的`limit`选项是一个方式。为了简单起见，systemtap 拒绝在使用 foreach 的时候进行数组修改

```
foreach (x = [a,b] in foo) { fuss_with(x) } simple loop in arbitrary sequence
foreach ([a,b] in foo+ limit 5) { } loop in increasing sequence of value, stop
after 5
foreach ([a-,b] in foo) { } loop in decreasing sequence of first key
```

## 3.5 Aggregates

前面我说值只有字符串和数值，我们有点小小当欺骗。有第三种类型：statistics aggregates，简称 aggregates。
这个类型值，通常用来收集数值上当统计信息，这个方式可以快速累加数据（不需要排他锁）以及很大的卷（仅存储 aggregated 流统计）。这个类型仅在全局变量中有效，可能独立存储，也可能在数组中存储。

添加一个值到 statistics aggregates，systemtap 使用特殊的操作符`<<<`，把这个想象成 C++的`<<`输出操作符: 左边的对象累计右边的数据。这个
操作很有效（使用共享锁），因为 aggregate 值在不同的处理中是独立的，仅在请求的时候处理。

```
a <<< delta_timestamp
writes[execname()] <<< count
```

读取 aggregate 值，使用特别的函数提取选择的统计能力。aggregates 值不能直接通过名字类似普通变量那样来读取。这些操作在全局需要一个排他锁，
因此相对是比较少见的。这些函数有：`@min`,`@max`, `@count`, `@avg`, `@sum`来获取单个值，此外，数据流的直方图可以通过`@hist_log`和`@hist_linear`来提取。这些对但哥数组排序但提取仅在打印的时候存在。

```
@avg(a) the average of all the values accumulated into a
print(@hist_linear(a,0,100,10)) print an “ascii art” linear histogram of the same data
stream, bounds 0 . . . 100, bucket width is 10
@count(writes["zsh"]) the number of times “zsh” ran the probe handler
print(@hist_log(writes["zsh"])) print an “ascii art” logarithmic histogram of the same
data stream
```

## 3.6 安全

脚本语言 关于安全性的问题，下面是`Q&A`:

**如果无穷循环，递归怎么办？** 探针 handler 有边界时间，systemtap 生成的 C 代码包含了显示检查，限制全部语句的执行时间在一个很小的数值内。类似的限制也体现在函数调用嵌套深度上。当超过限制时，探针 handler 清理异常，发出错误。systemtap 会话一般会配置在整个时间段配置一个异常。

**内存耗尽了怎么办？** 在执行探针 handler 的时候没有动态内存申请，初始化时会申请数组，函数上下文及缓存，这些资源可能在运行一个会话的时候，会耗尽，并产生错误结果。

**锁了怎么办？** 如果多个探针对同一个全局变量发生锁冲突，一个或多个探针 handler 超时了或者被取消了，这样对时间被记录为"skipped"探针，在会话结束是显示计数。一个可配置的 skipped 探针数字可以在会话中生成一个 abort。

**空指针，除 0 错误怎么办？** systemtap 生成的 C 代码存在潜在的危险操作会在运行时被检查，当无效的时候，会发出错误。很多算术运算和字符串操作在结果超出表达的限制时会安静地溢出。

**如果翻译或者编译的时候出了 bug 怎么办？** 当翻译的是欧出了 bug，或者运行时层已经存在，我们的测试套件会给一些保障。通过-p3 选项，可以查看生成整个 C 代码的过程，与内核相比，整体上，systemtap 不太关心编译器 bug。换句话说，如果构建内核是足够可信的，那么构建 systemtap 模块也应当如此。

**是全部事实么？** 在实践中，撰写本文时，systemtap 和潜在的 kprobe 系统存在一些弱点。在过去，将 probe 无差别地用在不常用地敏感地内核部分（低级地上下文切换，中断分发）已经有报告异常了。在发现这些 bugs 地时候我们已经修订了，并且构建了探针点的黑名单，但目前还不够全。

## 3.7 练习

1. 修改`timer-jiffies.stp`中最后的 probe，重置计数值，替换退出的代码修改为 reporting，
2. 编写一个脚本，每 10 秒，显示在时间段中使用系统调用最多的 5 个用户
3. 编写一个脚本，测试每个 cpu 上 get_cycles()计数的速度
4. 使用合适的 probe 获取 CPU 使用下大概的的 profile，每个 CPU 中 processes/users 的使用率
