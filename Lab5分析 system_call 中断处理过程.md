# 分析 system_call 中断处理过程

## MenuOS迁移到RISC-V架构

[MenuOS](https://github.com/mengning/menu/tree/master) 是接下来四次实验的平台。但是由于MenuOS原本设计是为x86架构所服务的，如果需要在RISC-V架构上的机器上运行，就必须对关键部分的汇编代码进行修改。

首先使用以下命令克隆[MenuOS](https://github.com/mengning/menu/tree/master)：

```bash
cd ~/riscv64_oslab/
mkdir MenuOS
cd MenuOS
git clone https://github.com/mengning/menu.git
```

克隆后，进入menu目录，使用你喜欢的编辑器对其中的TimeAsm函数进行修改，修改内容如下：

```c
int TimeAsm(int argc, char *argv[])
{
    time_t tt;
    struct tm *t;
    asm volatile(
        "li a0,201\n\t"
        "ecall \n\t"
        "sd a0, %0\n\t"
        : "=m" (tt)
     );
    t = localtime(&tt);
    printf("time:%d:%d:%d:%d:%d:%d\n",t->tm_year+1900, t->tm_mon, t->tm_mday, t->tm_hour, t->tm_min, t->tm_sec);
    return 0;
}
```

另外一个需要修改的文件是Makefile，这个文件的修改非常重要，将决定产生的什么架构的执行文件。

同样，请使用你喜欢的编辑器，打开Makefile，将其修改为以下内容：

```makefile
#
# Makefile for Menu Program
#

CC_PTHREAD_FLAGS			 = -lpthread
CC_FLAGS                     = -c 
CC_OUTPUT_FLAGS				 = -o
CC                           = riscv64-linux-gcc
RM                           = rm
RM_FLAGS                     = -f

TARGET  =   test
OBJS    =   linktable.o  menu.o test.o

all:	$(OBJS)
	$(CC) $(CC_OUTPUT_FLAGS) $(TARGET) $(OBJS) 
rootfs:
	riscv64-linux-gcc -o init linktable.c menu.c test.c  -static -lpthread
	riscv64-linux-gcc -o hello hello.c -static
	find init hello | cpio -o -Hnewc |gzip -9 > ../rootfs.img
	qemu-system-riscv64 -M virt \
			    -kernel ../linux-5.19.16/arch/riscv/boot/Image \
		        -initrd ../rootfs.img \
                -nographic
.c.o:
	$(CC) $(CC_FLAGS) $<

clean:
	$(RM) $(RM_FLAGS)  $(OBJS) $(TARGET) *.bak
```

需要注意的是，本次实验需要使用 `qemu-system-riscv64` 和 `qemu-system-riscv64` , 请完成第三次实验后，再来做此次实验。

使用以下命令进行编译和运行：

```bash
make -j$(nproc)
make rootfs -j$(nproc)
```

成功运行后，将会显示以下信息：

![image-20231105212015685](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105212015685.png)

## CWrite和的编写

现在我们在menu目录下, 使用你喜欢的编辑器修改test.c。这里使用神之编辑器emacs进行修改。

```bash
emacs test.c
```

打开之后，添加以下内容：

CWrite函数内容如下：

```C
int CWrite(void){
  char s[]="hello, world\n";
	write(1,s,13);
	return 0;
  return 0;
}
```

WriteAsm函数如下：

```c
int WriteAsm(void){
  char s[] = "hello, world\n";

    __asm__ volatile(
		    "li a2, 13\n"
		    "li a0, 1\n"
		    "mv a1, %[str]\n"
	    	"li a7, 64\n"
		    "ecall \n"
		    :
		    : [str] "r" (s)
    );
  return 0;
}
```

同样使用以下命令进行编译和运行：

```bash
make -j$(nproc)
make rootfs -j$(nproc)
```

启动MenuOS后：

输入 `help` 后回车，显示如下信息：

![image-20231105212850985](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105212850985.png)

再次输入 `write` 后回车，MenuOS输出以下信息：

![image-20231105212959159](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105212959159.png)

输入 `write-asm` 后回车，MenuOS输出以下信息：

![image-20231105213246034](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105213246034.png)

## GDB调试sys_write函数

使用你喜欢的编辑器在menu目录下，编写`start-gdb.sh` ，这里依旧使用emacs。

`start-gdb.sh` 的内容如下：

```shell
#!/bin/sh

qemu-system-riscv64 -M virt \
			    -kernel ../linux-5.19.16/arch/riscv/boot/Image \
		        -initrd ../rootfs.img \
                -nographic \
                -s -S
```

使用以下命令赋予`start-gdb.sh` 执行的权限：

```bash
chmod +x start-gdb.sh
```

使用以下命令启动 `start-gdb.sh`：

```bash
./start-gdb.sh
```

启动后，shell中应当不显示任何内容，如下所示：

![image-20231105221503598](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105221503598.png)

另开一个终端，进入MenuOS的目录下，使用以下命令启动 `gdb-multiarch ` 。

```bash
gdb-multiarch linux-5.19.16/vmlinux
```

启动之后，输入 `target remote:1234` 建立连接，显示如下信息说明连接建立完成：

![image-20231105221724247](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105221724247.png)

依次使用以下命令进行 `sys_write` 的调试：

```gdb
b sys_write
c
layout split
```

最终界面将显示如下信息：

![image-20231105221933745](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105221933745.png)

此时，将启动 `gdb-multiarch` 的终端和启动 `./start-gdb.sh` 的终端分别分屏左右，方便查看调试过程中的程序的输出，如下图所示：

![image-20231106161606896](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231106161606896.png)

执行上面打断点的操作后，在右侧图中的的最后一行显示：`[    0.507884] Run /init as init process` 说明Linux 内核已经初始化完成。
现在使用 十一次 `c` 命令，使得MenuOS加载到shell，如下图所示：

![image-20231106162535743](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231106162535743.png)

现在在MenuOS中输入 ` write-asm` 回车之后, MenuOS将会暂停输出。然后在左侧gdb窗口中输入命令 `c` ，MenuOS将会显示以下信息：

![image-20231106162854338](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231106162854338.png)

此时可以开始分析 `Cwrite` 中使用系统调用 `write` 的过程。首先使用以下命令对 `handle_exception` 打断点：

```c
b handle_exception
```

然后使用命令 `c` 进入到 `handle_exception` 断点处。在 `handle_exception` 函数中，根据异常号判断是否是系统调用。如果是系统调用，控制权会传递到系统调用处理的分发函数。

![image-20231111184909752](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231111184909752.png)

`generic_handle_arch_irq` 是 Linux 内核中的一个通用中断处理函数，它用于处理系统中断的基本工作。这个函数的主要作用是将控制权传递给适当的中断处理函数。在 Linux 内核中，每个中断都有一个对应的中断处理函数，用于处理特定类型的中断。`generic_handle_arch_irq` 通过查询中断描述符表（Interrupt Descriptor Table，IDT）来确定要执行的中断处理函数，并将控制权转交给它。

使用命令 `c` 进入  `ksys_write` 断点处。在 `SYSCALL_DEFINE3` 的函数中的返回语句 `return ksys_write(fd, buf, count); ` ，如下图所示：

![image-20231106162942747](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231106162942747.png)

继续使用 `si` 命令，将程序将执行下一条汇编语句，如下图所示：

![image-20231106163210213](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231106163210213.png)

`system_call` 调用过程如下：

```mermaid
graph TD;
   ecall指令触发中断-->handle_exception处理中断;
   handle_exception处理中断--> handle_syscall处理系统调用;
   handle_syscall处理系统调用-->系统调用sys_write;
   系统调用sys_write--> ret_from_exception中断返回:
```
#### 具体的流程
由于一些限制，光靠这个gdb无法明确的知道系统调用的一些细节，在这里展开讲解一些。
  - ecall触发trap，ecall它其实是跳转到一个地址，这个地址被存在stvec寄存器中（请了解[RISCV特权架构指令](http://riscvbook.com/chinese/RISC-V-Reader-Chinese-v2p1.pdf)）
  - 这个stvec的地址在操作系统的启动的时候就已经被设置好了在`arch/riscv/kernel/head.S `里面，它就是handle_exception。
  - handle_execption会保存用户的上下文，然后把内核上下文写入寄存器（包括页表）
  - sstatuc、sepc、stval、scause、sscratch 这 5 个 csr 寄存器（请了解[RISCV特权架构指令](http://riscvbook.com/chinese/RISC-V-Reader-Chinese-v2p1.pdf)）存储了触发trap的信息。
  - scause里面存储的是出发trap的信息，scause 寄存器最高位含义如下：最高位=1：interrupts 最高位=0：exceptions，如scause的值大于0那么就是由excptions触发，如果小于0，那么就是一个中断。我们这里是大于0的。
  - 接下里判断scause是否等于EXC_SYSCALL 这个值是8 ，也就是当 scause==8 时，就表示由 sys_call 触发的 trap,从而进入handle_syscall
  - handle_syscall 里面会将spec也就是出错的地址（我们这里是ecall的地址）+4并且存储下来，用于正确返回（sret的时候是ret到下一个ecall的地址防止无限循环），然后根据a7里面的值确定是哪个系统调用。找到相应的系统调用,返回。注意在 handle_syscall 里跳转到实际的系统调用函数时把返回地址设置成了 ret_from_syscall。所以上述的 sys_write 函数返回后会跳转到 ret_from_syscall 继续执行。
  - ret_from_syscall会将保存的用户模式的上下文中的a0（返回值）切换为系统调用运行之后的返回值，然后恢复现场，最后使用sret来回到用户态