# 理解进程调度时机跟踪分析进程调度与进程切换的过程

## 实验要求： 

1. 理解 Linux 系统中进程调度的时机

2. 使用 gdb 跟踪分析一个 schedule()函数 

3. 仔细分析 switch_to 中的汇编代码，理解进程上下文的切换机制，以及与中断上下文切换的关系；

## 理解 Linux 系统中进程调度的时机

进程是计算机中运行的程序的抽象，它包含了程序代码、数据、以及程序执行的上下文。进程调度的策略和算法决定哪个进程应该在某个时刻运行。进程调度的时机大多和中断有关。在多任务操作系统中，中断是一种机制，它允许操作系统在发生某些事件时打断正在执行的进程，转而执行另一个进程。这种打断和切换的机制是进程调度的基础——中断上下文恢复。中断上下文恢复是指在中断服务例程（ISR）中，由于中断事件而中断的正在执行的任务（通常是用户进程）的上下文（包括寄存器状态、程序计数器等）被保存起来，以便在中断处理完成后能够正确地恢复原来的任务继续执行。

## gdb 跟踪分析 schedule()函数

在menuos的目录之下，启动 `init_gdb.sh` 和 `star-gdb.sh` 两个bash脚本和gdb远程连接后，在gdb中输入 `b schedule` 命令进行对schedule函数进行打断点后，使用`c` 命令运行到断点处。如下图所示：

![image-20231109201036323](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109201036323.png)

在RISC-V架构下的Linux内核中，schedule断点并没有出现在 `kernel/sched/core.c` 中而是出现在 `arch/riscv/include/asm/current.h` 中，并指向 `riscv_current_is_tp` 。

![image-20231109201842239](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109201842239.png)

在进行下一步后，断点跳至 `kernel/sched/core.c` 中的 `schedule` 函数中：

![image-20231109202045031](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109202045031.png)

在 `tsk` 变量赋值后， 使用 `p *tsk` 命令查看 `tsk` 的内容：

![image-20231109202345331](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109202345331.png)

使用 `b __schedule` 对 `__schedule` 设置断点并使用 `c` 命令运行到断点处：

![image-20231109202607880](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109202607880.png)

在 `__schedule` 函数中使用 `n` 命令单独后，发现 `__schedule` 函数调用了 `context_switch` 函数：

![image-20231109202816225](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109202816225.png)

对 `context_switch` 打入断点后，单步执行后发现调用 `switch_to`：

![image-20231109203216397](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109203216397.png)

此时无法对 `switch_to ` 进行打断点，因为 `switch-to` 是一个宏定义，在预处理阶段就已经被替换了，所以只能对 `__switch_to` 打断点，由于RISC-V架构下 `__switch_to` 是有纯汇编写的，所以无法展示c语言代码。如下图所示：

![image-20231109204506752](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109204506752.png)

## 分析 switch_to 中的汇编代码(王瑞 todo)

在RISC-V架构的Linux内核中，switch_to的汇编代码位于 `arch/riscv/kernel/entry.S` 中。

以下代码是switch_to部分的汇编代码：

```asm
ENTRY(__switch_to)
	/* Save context into prev->thread */
	li    a4,  TASK_THREAD_RA
	add   a3, a0, a4
	add   a4, a1, a4
	REG_S ra,  TASK_THREAD_RA_RA(a3)
	REG_S sp,  TASK_THREAD_SP_RA(a3)
	REG_S s0,  TASK_THREAD_S0_RA(a3)
	REG_S s1,  TASK_THREAD_S1_RA(a3)
	REG_S s2,  TASK_THREAD_S2_RA(a3)
	REG_S s3,  TASK_THREAD_S3_RA(a3)
	REG_S s4,  TASK_THREAD_S4_RA(a3)
	REG_S s5,  TASK_THREAD_S5_RA(a3)
	REG_S s6,  TASK_THREAD_S6_RA(a3)
	REG_S s7,  TASK_THREAD_S7_RA(a3)
	REG_S s8,  TASK_THREAD_S8_RA(a3)
	REG_S s9,  TASK_THREAD_S9_RA(a3)
	REG_S s10, TASK_THREAD_S10_RA(a3)
	REG_S s11, TASK_THREAD_S11_RA(a3)
	/* Restore context from next->thread */
	REG_L ra,  TASK_THREAD_RA_RA(a4)
	REG_L sp,  TASK_THREAD_SP_RA(a4)
	REG_L s0,  TASK_THREAD_S0_RA(a4)
	REG_L s1,  TASK_THREAD_S1_RA(a4)
	REG_L s2,  TASK_THREAD_S2_RA(a4)
	REG_L s3,  TASK_THREAD_S3_RA(a4)
	REG_L s4,  TASK_THREAD_S4_RA(a4)
	REG_L s5,  TASK_THREAD_S5_RA(a4)
	REG_L s6,  TASK_THREAD_S6_RA(a4)
	REG_L s7,  TASK_THREAD_S7_RA(a4)
	REG_L s8,  TASK_THREAD_S8_RA(a4)
	REG_L s9,  TASK_THREAD_S9_RA(a4)
	REG_L s10, TASK_THREAD_S10_RA(a4)
	REG_L s11, TASK_THREAD_S11_RA(a4)
	/* The offset of thread_info in task_struct is zero. */
	move tp, a1
	ret
ENDPROC(__switch_to)
```

## 理解进程上下文的切换机制和中断上下文切换的关系

进程上下文切换是指挂起现在正在运行的进程，并恢复之前挂起的某个进程。恢复一个之前挂起的进程必须使得寄存器保存现在被挂起的进程的值，而这部分需要保存的值被称为进程上下文。

进程执行的环境切换分为两步：

1. 从就绪队列中选择一个进程
2. 完成进程的上下文切换

这两个步骤分别涉及 `pick_next_task` 和 `context_witch` 两个函数。

中断是一种异步事件，可以打断当前正在执行的任务，跳转到中断服务例程（ISR）来处理中断。在RISC-V架构中，中断通常被称为"异常"（exceptions）。RISC-V将中断、陷阱（traps）、系统调用（system calls）等都统一称为异常。

RISC-V的异常处理机制包括三种类型的异常：

1. **中断（Interrupts）：** 由外部事件引起，例如时钟中断、I/O中断等。
2. **陷阱（Traps）：** 由程序内部事件引起，例如软件陷阱，通常用于实现系统调用。
3. **故障（Faults）：** 由于非法操作或错误引起的异常，例如页错误。

中断上下文切换是指在处理中断时，保存当前任务的上下文，执行中断服务例程，然后在处理完成后，恢复原任务的上下文。

进程上下文的切换机制和中断上下文切换的相似之处：

1. 都需要停止当前进程并保存当前停止的任务的上下文
2. 保存当前任务的上下文后，都需要切换上下文
3. 保存的上下文，在将来的某个时段都会重新恢复运行

不同点：

1. **触发时机不同：** 进程上下文切换通常是由操作系统的调度器在特定时机决定的，例如时钟中断、I/O中断等。而中断上下文切换是由硬件中断信号触发的，例如定时器中断、硬件故障中断等。
2. **执行过程不同：** 进程上下文切换包括保存和加载两个阶段，而中断上下文切换主要涉及中断服务例程的执行过程。
3. **保存和恢复的内容不同：** 进程上下文切换需要保存和恢复整个进程的执行环境，包括寄存器、内存页表等。中断上下文切换主要关注于保存和恢复与中断服务例程执行相关的上下文。

