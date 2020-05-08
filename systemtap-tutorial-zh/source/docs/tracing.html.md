---

layout: "docs"
page_title: "Tracing"
sidebar_current: "systemtap"
description: |-
本文描述systemtap怎么进行跟踪
---

最简单的 probe 类型是跟踪单个事件。在程序中插入 print 语句即可，通常这是解决问题的第一步：通过查看发生的历史信息来勘查。

```stap
probe syscall.open
{
  printf("%s(%d) open(%s)\n", execname(), pid(), argstr)
}

# after 4 seconds
probe timer.ms(4000)
{
  exit()
}
```

```bash
# stap strace-open.stp
vmware-guestd(2206) open ("/etc/redhat-release", O_RDONLY)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
df(3433) open ("/etc/ld.so.cache", O_RDONLY)
df(3433) open ("/lib/tls/libc.so.6", O_RDONLY)
df(3433) open ("/etc/mtab", O_RDONLY)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
```

This style of instrumentation is the simplest. 仅需要告知 systemtap 打印每个事件的内容。为在脚本语言中表述上述事情，需要
在脚本中确定哪里该探测，以及怎么打印。

## 2.1 探测哪里

Systemtap 支持一系列内置事件。systemtap 携带了脚本库，每个脚本称为 tapset，这些脚本可能在内置族中定义了额外的内容。在`stapprobes`手册页中
查看这些信息的详情和其他许多探针点族。所有这些事件使用统一的语法命名--使用点分隔的参数标识符：

```
begin           systemtap会话的开始
end             systemtap会话的结束
kernel.function("sys_open") 内核中函数名为sys_open的入口
syscall.close.return 系统调用close的返回
module("ext3").statement(0xdeadbeef) 在ext3的文件系统驱动中的地址指令
timer.ms(200) 每200毫秒触发一次的timer
timer.profile 每个CPU周期触发一次的timer
perf.hw.cache_misses CPU缓存失效的数量
procfs("status").read 一个试图读取synthetic file的进程
process("a.out").statement("*@main.c:200") a.out程序中第200行
```

我们试着跟踪一个内核中`net/socket.c`源文件的所有的函数的入口和退出。`kernel.function`探针会让你处理这个更简单一些，由于 systemtap 检查内核调试信息用于将目标代码和源代码关联起来。类似 debugger 那样的处理方式：如果可以命名或者指定位置，你就可以探测。使用 `kernel.function("*@net/socket.c").call`作为函数入口，使用`kernel.function("*@net/socket.c").return`作为函数退出。注意在函数名称中使用通配符，然后跟随`@FILENAME`的用法。如果希望更精确地限制搜索，也可以将通配符放在文件名中，然后添加一个冒号(:)以及行号。当 systemtap 会把每个地方设置的探针匹配探针入口，一些通配符可以扩展为成百上千个探针，因此你需要小心处理。

一旦识别探针入口，systemtap 的骨架脚本就会出来了。关键字 `probe` 申明了一个或多个（通过逗号分隔）探针点，之后的`{}`则包含了谈征需要处理的整个 handler。

```
probe kernel.function("*@net/socket.c") { }
probe kernel.function("*@net/socket.c").return { }
```

可以运行类似上面的脚本， 空的 handler 没有任何输出。将上面两行存入文件，使用`stap -v FILE`方式运行，使用`^C`退出（-v 选项表示在处理时打印更多的调试信息，-h 查看更多的选项）

## 2.2 打印什么

当对每个函数的入口和退出感兴趣的时候，则应该打印包含的函数名。当其他跟踪函数嵌套深入调用时可以更好的被读取，systemtap 应该将行缩进。为了区分单个处理中可能是并发运行的情况，systemtap 应该在行中打印进程 ID。

systemtap 提供了一系列上下文数据，用来作为格式化处理。通常以函数形式在 handler 出现，这些函数见`function::*`手册，更多的定义在 tapset 库中，下面是一些示例：

```
tid() 当前线程的id
pid() 当前线程下的进程（任务组）id
uid() 当前用户
execname() 当前进程名称
cpu() 当前cpu编号
gettimeofday_s() 以epoch的秒数
get_cycles() 硬件cycle counter的快照
pp() 当前被处理的探针点的描述字符串
ppfunc() 哪个探针被处理的函数名，？应该可能是可以为空的？
$$vars 当存在时，所有当前局部变量的pretty-print
print_backtrace() 如果可以，打印内核的backtrace
print_ubacktrace() 如果可以，打印用户空间的backtrace
```

返回值可能是字符串，也可能是数字，内置的`print`函数接受单个参数，或者使用内置的 c 样式的`printf`函数，可以通过`%s`表示字符串，`%d`表示数字来格式化参数。`printf`和其他函数，通过逗号来分隔参数。别忘记在格式化尾部添加`"\n"`。系统提供了更多的打印和格式化函数。

在 tabset 库中特别的可以处理的函数是`thread_indent`，通过给定缩进的偏移参数，会在每个线程(`tid()`)中存储内部缩进计数，返回一个普通的跟踪数据，尾部跟随合适的缩进空格数。普通的跟踪数据由时间戳（从线程初始化开始的毫秒数），进程名以及线程 id 组成。这个方式下，就可以知道是调用了什么函数，谁调用了，花了多久。

下面的脚本显示了调用的方式，脚本缺少了`exit()`函数，需要通过`^C`来中断脚本

```
probe kernel.function("*@net/socket.c").call {
printf ("%s -> %s\n", thread_indent(1), ppfunc())
}
probe kernel.function("*@net/socket.c").return {
printf ("%s <- %s\n", thread_indent(-1), ppfunc())
}
```

```bash
stap socket-trace.stp
0 hald(2632): -> sock_poll
28 hald(2632): <- sock_poll
[...]
0 ftp(7223): -> sys_socketcall
1159 ftp(7223): -> sys_socket
2173 ftp(7223): -> __sock_create
2286 ftp(7223): -> sock_alloc_inode
2737 ftp(7223): <- sock_alloc_inode
3349 ftp(7223): -> sock_alloc
3389 ftp(7223): <- sock_alloc
3417 ftp(7223): <- __sock_create
4117 ftp(7223): -> sock_create
4160 ftp(7223): <- sock_create
4301 ftp(7223): -> sock_map_fd
4644 ftp(7223): -> sock_map_file
4699 ftp(7223): <- sock_map_file
4715 ftp(7223): <- sock_map_fd
4732 ftp(7223): <- sys_socket
4775 ftp(7223): <- sys_socketcall
```

## 2.3 实践

1. 使用 -L 选贤参数列出所有含 `nit` 名称的内核函数
2. 跟踪一些系统调用（使用 `syscall.NAME` 和`.return` 探针点），类似上面的`thread_indent`探针处理，使用`$$params`和`$$returns`打印参数，并解释结果
3. 修改上面的脚本，移除第一个探针中的`.call`修饰，注意函数的入口和函数的返回是怎么不能匹配的。其因为在于第一个探针不能匹配普通的函数入口和内联的函数。然后将`.call`修饰放回去，添加另外一个探针`probe kernel.function("*@net/socket.c").inline`，在探针 handler 中使用`printf`语句，显示在`.call`和`.return`线程缩进中内联函数的输出？
