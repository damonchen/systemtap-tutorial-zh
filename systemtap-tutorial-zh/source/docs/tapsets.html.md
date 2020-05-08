---

layout: "docs"
page_title: "Tapset"
sidebar_current: "systemtap"
description: |-
本文描述systemtap下的tapset信息
---

# 4 Tapset

在编写合适的脚本后，你将成为同事中的专家，并且希望使用你的脚本。Systemtap 在控制模式下很容易共享，构建一个脚本需要使用的库，事实上，上面脚本中使用的所有函数（如 pid())，都是来自于 tabset 的脚本。安装在指定目录下的 tapset 是一个设计为重用的脚本。

## 4.1. 自动选择

Systemtap 通过在 tabset 库语义搜索脚本定义的的符号，来解决在脚本中未定义的全局符号(探针，函数变量）的引用。Tapset 脚本安装在`/usr/share/systemtap/tapset`目录下，用户可以通过额外的`-I DIR`选项来添加目录，Systemtap 会在这些目录下查找`.stp`结尾的脚本。

对特定内核版本或架构下搜索这些子目录有一个特别规定，即一个名称必须大于 kernel faimilies。一般说，搜索顺序从特殊到一般。

```bash
# stap -p1 -vv -e ’probe begin { }’ > /dev/null
Created temporary directory "/tmp/staplnEBh7"
Searched ’/usr/share/systemtap/tapset/2.6.15/i686/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/2.6.15/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/2.6/i686/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/2.6/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/i686/*.stp’, match count 1
Searched ’/usr/share/systemtap/tapset/*.stp’, match count 12
Pass 1: parsed user script and 13 library script(s) in 350usr/10sys/375real ms.
Running rm -rf /tmp/staplnEBh7
```

在一个未定义的符号在脚本文件被找到，整个文件就会被加入到探针 session 的分析中。在没有文件满足上述的过程时，这个搜索过程会一直重复持续。当有任何一个符号没找到当时候，systemstap 会发出一个 signal。

允许只为适用的内核版本/体系结构对定义一些全局符号，在不适用的及其上使用符号会引起错误。简单说，在不同内核上的同一个符号可以定义为不同的形式，大致相同的方式是不同的 kernel
`include/asm/ARCH`文件包含了宏用来提供了移植层。

另外一种用法是从 tapset 但 routine 中分隔默认参数作为实现，例如，考虑 定义了关于处理调度的时间耗时间隔的代码的 tapset。数据收集的代码可以一般可以用时间单位（jiffies, wall-clock 秒,cycle counts)。通常应该有一个默认值，并且不要用户选择，不需要额外的运行时检查。

一个 tabpset 通过韩说或者探测点别名（后面有描述）暴露有效的数据，全局数据可以被计算，并保持通过探针内部保持更新。任何对全局变量的外部引用都将偶然激活所有必需的探针。

```bash
# cat tapset/time-common.stp
global __time_vars
function timer_begin (name) { __time_vars[name] = __time_value () }
function timer_end (name) { return __time_value() - __time_vars[name] }
# cat tapset/time-default.stp
function __time_value () { return gettimeofday_us () }
# cat tapset-time-user.stp
probe begin
{
    timer_begin ("bench")
    for (i=0; i<100; i++) ;
    printf ("%d cycles\n", timer_end ("bench"))
    exit ()
}
function __time_value () { return get_ticks () } # override for greater precision
```

## 4.2 探测点别名

探测点别名允许从已有的探针中创建新的探针，通常在新的探针提供更高级的抽象中有用。例如，系统调用 tapset 从`syscall.open`形式中定义探测点别名，是底层`kernel.function("sys_open")`的别名。即便将来内核重新定义了 sysopen，别名仍旧可以被使用。

探测点别名定义类似定义普通的探针，都是以`probe`关键字开头，由一个探针 handler 语句块结束。别名通过赋值号来描述探测点别名。命名新探测点的另一个探测将创建一个实际的探测，并带有别名的处理程序。

前置轻微有几个目标，允许别名定义为在探针传递给用户指定的 handler 前预处理上下文，这个有几个不同的形式：

```
if ($flag1 != $flag2) next skip probe unless given condition is met
name = "foo" supply probe-describing values
var = $var extract target variable to plain local variable
```

下面表示了探针点使用的别名定义，表示单个探测点别名如何展开多个探测点及其他别名，也支持探测点`*`。这些功能旨在合理组合。

```stap
probe syscallgroup.io = syscall.open, syscall.close,
                syscall.read, syscall.write
    { groupname = "io" }
probe syscallgroup.process = syscall.fork, syscall.execve
    { groupname = "process" }

probe syscallgroup.*
{ groups [execname() . "/" . groupname] ++ }

probe end
{
    foreach (eg+ in groups)
        printf ("%s: %d\n", eg, groups[eg])
}
global groups
```

```bash
# stap probe-alias.stp
05-wait_for_sys/io: 19
10-udev.hotplug/io: 17
20-hal.hotplug/io: 12
X/io: 73
apcsmart/io: 59
[...]
make/io: 515
make/process: 16
[...]
xfce-mcs-manage/io: 3
xfdesktop/io: 5
[...]
xmms/io: 7070
zsh/io: 78
zsh/process: 5
```

## 4.3 嵌入 C

有时候，tapset 需要一些内核的数据值，但这些不能够从普通但目标变量中提取，这是由于这些数据值可能复杂但数据结构，可能需要注意锁或者通过红来定义但。systemptap 提供了"逃生窗口"，可以超越语言安全地使用。在某个上下文中，可以在 tapset 中嵌入原生的 C，交换能力以获取第 3.6 节中列出的安全保证。终端用户脚本一般不需要包含嵌入 C 代码，除非 systemtap 使用-g（guru 模式）选项运行。tapset 脚本在特权模式下自动获得 guru 模式。

嵌入 C 可以有一个脚本的函数体，使用`%{`和`%}`来替换`{`和`}`。任何 C 代码都会被转换为内核模块：让代码变得安全和正确。为了获取参数及返回值，需要使用宏`STAP_ARG_*`和`STAP_RETVALUE`。数据采集函数族`pid()`, `execname()`以及其他类似的都是嵌入 C 代码。

由于 systemtap 不能够检查 C 代码推导类型，可选的标记语法可以帮助类型推导处理，在参数名或函数名尾部添加`:string`或`:long`来表示字符串或数值类型。此外，为了描述类似的声明代码`#include <linux/foo.h>`，脚本可能在`%{%}`块之外。这个方式允许嵌入 C 代码引用一般内核的数据类型

嵌入 C 代码中有一些安全相关的限制应该被开发者所知道：

1. 不要对未知的或测试无效的指针解引用
2. 不要调用任何内核过程，这个可能会导致休眠或者异常
3. 考虑可能的不希望的递归，嵌入 C 函数调用的例程可能是探针的主题，如果探针 handler 可以调用嵌入 C 函数，可能会无穷的
4. 如果锁数据结构是必须的，使用`trylock`类型去获取锁，如果获取失败，则放弃，不会锁。

## 4.4 惯用命名

上述的 tapset 搜索机制，可能在单个会话中会发现很多潜在的脚本。这个会导致名字冲突，在不同的 tapset 中偶尔会出现使用同样的函数名或全局变量名，这个会导致在翻译时或者运行时发生错误。

控制这些问题，systemtap tapset 开发这需要注意下面的惯用命名，如下：

1. 取一个独一无二的 tapset 名称，替换下面的 TAPSET
2. 分割标识符意味着使用 tapset 的用户来自于内部实现
3. 在相应的手册页记录第一组
4. 如果可能与其他 tapset 或终端用户的标识符冲突，则使用`TAPSET`作为外部标识符的前缀
5. 用合适的前缀处理任何探测点别名
6. 用`__TAPSET__`的作为内部标识符的前缀

```
%{
#include <linux/sched.h>
#include <linux/list.h>
%}



# definition function with type annoation
function task_execname_by_pid:string(pid:long) %{
    struct task_struct *p;
    struct list_head *_p, *_n;

    list_for_each_safe(_p, _n, &current->tasks) {
        p = list_entry(_p, struct task_struct, tasks);
        if (p->pid == (int)STAP_ARG_pid) {
            snprintf(STAP_RETVALUE, MAXSTRINGLEN, "%s", p->comm);
        }
    }
%}


probe begin
{
    printf("%s(%d)\n", task_execname_by_pid(target()), target())
    exit()
}
```

## 4.5 练习

1. 编写 tapset 实现延迟并"cancelable"日志，导出封装文本字符串为私有数组的函数，该返回 id 标识。定时刷新数组到标准日志输出的含基于定时的探针。如果入口还没有被刷新，则导出另一个函数，允许从队列中取消文本字符串，有人可能会猜测类似功能和分解器存在。
2. 使用函数返回与时间戳中相同的值来创建"relative timestamp"的 tapset，相对于脚本开始时间的处理
3. 创建一个 tapset，导出含最新进程 id 映射到进程名的全局的数组。拦截系统调用(execve)更新列表
4. 发送 tapset 想法到邮件列表
