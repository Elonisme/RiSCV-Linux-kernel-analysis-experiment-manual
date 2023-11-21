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

## 分析 switch_to 中的汇编代码

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

## 分析汇编代码
看到这段汇编代码，我们首先要做的是预处理一下，首先理解这个REG...和TASH...这些宏是什么。这里可以通过全局搜索，找到他们的定义。经过全局搜索发现，他们被定义在`arch/riscv/include/asm/include.h`里面。  
```c
#ifndef _ASM_RISCV_ASM_H
#define _ASM_RISCV_ASM_H

#ifdef __ASSEMBLY__
#define __ASM_STR(x)	x
#else
#define __ASM_STR(x)	#x
#endif

#if __riscv_xlen == 64
#define __REG_SEL(a, b)	__ASM_STR(a)
#elif __riscv_xlen == 32
#define __REG_SEL(a, b)	__ASM_STR(b)
#else
#error "Unexpected __riscv_xlen"
#endif

#define REG_L		__REG_SEL(ld, lw)
#define REG_S		__REG_SEL(sd, sw)

``` 
在riscv中一个字是32位，64字是double world。在汇编代码中一般用w（32位）表示一个字，用d（64位）表示2个字。在c语言的宏里面#的作用是将传入的字符变成一个用双引号包裹的字符串（请自行查看相关标准）。我们这里不需要双引号，并且是64位，所以REG_L预处理之后应该是`ld`和`sd`

在arch/riscv/kernel/asm-offsets.c中我们可以找到TASK_THREAD_RA_RA的定义  
```c
void asm_offsets(void)
{
	OFFSET(TASK_THREAD_RA, task_struct, thread.ra);
	...
	DEFINE(TASK_THREAD_RA_RA,
		  offsetof(struct task_struct, thread.ra)
		- offsetof(struct task_struct, thread.ra)
	);
}
```
这个TASK_THREAD_RA_RA的意思就是得到ra寄存器距离ra寄存器的offset。这里是用task_struct里面的thread.ra的地址减去thrad.ra的地址。而TASH_THREAD_RA就是thread.ra距离task_struct的偏移地址。  
接下来我们来看看这个thread的数据结构,它在`/arch/riscv/include/asm/processor.h`里面。

```c
struct thread_struct {
	/* Callee-saved registers */
	unsigned long ra;
	unsigned long sp;	/* Kernel mode stack */
	unsigned long s[12];	/* s[0]: frame pointer */
	struct __riscv_d_ext_state fstate;
	unsigned long bad_cause;
	unsigned long vstate_ctrl;
	struct __riscv_v_ext_state vstate;
};
```
我们再来看看调用switch_to的代码，发现第一个参数是需要保存的tash_struct第二个参数是需要加载的tash_struct。  
也就是说a0 = prev ,a1 = next
```c
#define switch_to(prev, next, last)			\
do {							\
	struct task_struct *__prev = (prev);		\
	struct task_struct *__next = (next);		\
	if (has_fpu())					\
		__switch_to_aux(__prev, __next);	\
	((last) = __switch_to(__prev, __next));		\
} while (0)
```
让我们以伪代码的形式来理解这段代码。
```asm
a0 = prev
a1 = next 
offset = &task_struct - &(task_struct->ra)//ra距离task_struct偏移
ENTRY(__switch_to)
	/* Save context into prev->thread */
	li    a4,  offset
	add   a3, a0, a4 //a3 = prev->thread.ra
	add   a4, a1, a4 //a4 = next->thread.ra
	sd ra,  a3 
	sd sp,  8(a3)
	sd s0,  16(a3)
	sd s1,  24(a3)
	sd s2,  32(a3)
	sd s3,  40(a3)
	sd s4,  48(a3)
	sd s5,  56(a3)
	sd s6,  64(a3)
	sd s7,  72(a3)
	sd s8,  80(a3)
	sd s9,  88(a3)
	sd s10, 96(a3)
	sd s11, 104(a3)
	/* Restore context from next->thread */
	ld ra,  0(a4)
	ld sp,  8(a4)
	ld s0,  16(a4)
	ld s1,  24(a4)
	ld s2,  32(a4)
	ld s3,  40(a4)
	ld s4,  48(a4)
	ld s5,  56(a4)
	ld s6,  64(a4)
	ld s7,  72(a4)
	ld s8,  80(a4)
	ld s9,  88(a4)
	ld s10, 96(a4)
	ld s11, 108(a4)
	/* The offset of thread_info in task_struct is zero. */
	move tp, a1 // 将线程寄存里面呢的值改为当前的task_stuct的地址
	ret
ENDPROC(__switch_to)

```
转化为这种代码之后，我们就非常容易理解了，只要会读riscv的汇编（请参阅lab2）就能理解代码的意思。  
这里简单解释一下，就是将当前线程的ra，sp，s0-s11存入prev->thread部分。然后将next->thread里面的值加载进寄存器，最后切换线程指针tp。

为什么只需保存这么少？因为我们只需要保存caller save的寄存器就够了，其他的寄存器是不影响的。如果需要详细理解，请参阅lab2里面的链接

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

