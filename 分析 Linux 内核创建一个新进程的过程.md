# 分析 Linux 内核创建一个新进程的过程

阅读理解 task_struct 数据结构 https://github.com/torvalds/linux/blob/v3.18-rc6/include/linux/sched.h#L1235；

分析 fork 函数对应的内核处理过程 sys_clone，理解创建一个新进程如何创建和修改 task_struct 数据结构；

使用 gdb 跟踪分析一个 fork 系统调用内核处理函数 sys_clone ，验证您对 Linux 系统创建一个新进程的理解,推荐在实验楼 Linux 虚拟机环境下完成实验。 特别关注新进程是从哪里开始执行的？为什么从那里能顺利执行下去？即执行起点与内核堆栈如何保证一致。