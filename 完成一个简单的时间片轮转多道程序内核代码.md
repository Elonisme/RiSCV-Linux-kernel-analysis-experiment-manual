# 完成一个简单的时间片轮转多道程序内核代码

## 串口连接开发板

我们使用USB转TTY下载线连接Starfive旗下的VisionFive2开发板。主机操作系统是Ubuntu22.04，具体配置如下。

![image-20231029115426545](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029115426545.png)

引脚连接实物图如下：

![image-20231029115801584](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029115801584.png)



## mincom串口工具下载与设置

1. 下载

   ```bash
   sudo apt update
   sudo apt install minicom
   ```

2. 设置

   ```bash
   sudo minicom -s
   ```

进入minicom的设置界面后：方向键选择 Serial port setup后按回车确定。

![image-20231029120122847](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029120122847.png)

进入串口设置界面后：按Shift+a选择Serial Device,将设备名改为/dev/ttyUSB0.

![image-20231029120332044](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029120332044.png)

修改完成后按下回车，返回上级菜单后，选择Save setup as dfl保存配置。

![image-20231029120447877](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029120447877.png)

保存后，选择Exit退出Minicom

## 使用Minicom连接VisionFive2

首先将连接好引脚的USB插口插入笔记本上，然后使用以下命令启动Minicom。

```bash
sudo minicom
```

如果启动失败显示没有/dev/ttyUSB0，请参见下一节《CH340系列串口驱动占用》。

启动后会显示如下的信息：

![](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029120808567.png)

然后使用电源线连接开发板，给开发板加电。此时Minicom会开始显示加电后各种程序启动的信息：

![image-20231029121051751](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029121051751.png)



## CH340系列串口驱动占用

如果配置好Minicom后显示没有ttyUSB的问题，那么可能是由于CH340系列串口驱动被占用。

使用以下命令判断是否驱动被占用：

```bash
sudo dmesg | grep brltty
```

如果显示以下信息：

```bash
[ 7033.078452] usb 1-13: usbfs: interface 0 claimed by ch341 while &apos;brltty&apos; sets config #1
```

说明：驱动占用

解决方法：使用以下命令卸载brltty

```bash
sudo apt remove brltty
```

然后重新进行插拔USB接口，并使用以下的命令查看是否恢复正常：

```bash
ls /dev/ttyUSB0 
```

若显示：

```bash
ls /dev/ttyUSB0 
/dev/ttyUSB0
```

说明恢复正常，可以按照之前的步骤继续实验。

## 为编译Linux内核做准备

StarFive为VisionFive2提供了一个[GitHub仓库](https://github.com/starfive-tech/VisionFive2)，阅读这个仓库中的README文件是本次实验的一部分。从README得知，为连编译能够在开发板上运行的Linux内核，首先需要为编译提供依赖环境。

使用以下的命令，在ubuntu22.04上创建可实现交叉编译risc-v架构的环境依赖：

```bash
sudo apt update
sudo apt-get install build-essential automake libtool texinfo bison flex gawk g++ git xxd curl wget gdisk gperf cpio bc screen texinfo unzip libgmp-dev libmpfr-dev libmpc-dev libssl-dev libncurses-dev libglib2.0-dev libpixman-1-dev libyaml-dev patchutils python3-pip zlib1g-dev device-tree-compiler dosfstools mtools kpartx rsync
```

安装Git LFS:

```bash
curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
sudo apt-get install git-lfs
```

需要注意的是安装Git LFS需要访问外网，要完成此步骤需读者自行解决访问外网的问题或者使用国内镜像网站。

### 克隆[VisionFive2](https://github.com/starfive-tech/VisionFive2) 仓库

使用以下命令克隆VisionFive2:

```bash
git clone https://github.com/starfive-tech/VisionFive2.git
cd VisionFive2
git checkout JH7110_VisionFive2_devel
git submodule update --init --recursive
```

切换分支：

```bash
cd buildroot && git checkout --track origin/JH7110_VisionFive2_devel && cd ..
cd u-boot && git checkout --track origin/JH7110_VisionFive2_devel && cd ..
cd linux && git checkout --track origin/JH7110_VisionFive2_devel && cd ..
cd opensbi && git checkout master && cd ..
cd soft_3rdpart && git checkout JH7110_VisionFive2_devel && cd ..
```

## RISC-V架构MyKernel内核的构建（王瑞填写）

myinterrupt.c

```c
/*
 *  linux/mykernel/myinterrupt.c
 *
 *  Kernel internal my_timer_handler
 *
 *  Copyright (C) 2023, 2023  WangRui
 *
 */
#include <linux/types.h>
#include <linux/string.h>
#include <linux/ctype.h>
#include <linux/tty.h>
#include <linux/vmalloc.h>

#include "mypcb.h"

extern tPCB task[MAX_TASK_NUM];
extern tPCB * my_current_task;
extern volatile int my_need_sched;
volatile int time_count = 0;

/*
 * Called by timer interrupt.
 * it runs in the name of current running process,
 * so it use kernel stack of current running process
 */
void my_timer_handler(void)
{
    if(time_count%1000 == 0 && my_need_sched != 1)
    {
        printk(KERN_NOTICE ">>>my_timer_handler here<<<\n");
        my_need_sched = 1;
    } 
    time_count ++ ;  
    return;  	
}

void my_schedule(void)
{
    tPCB * next;
    tPCB * prev;

    if(my_current_task == NULL 
        || my_current_task->next == NULL)
    {
    	return;
    }
    printk(KERN_NOTICE ">>>my_schedule<<<\n");
    /* schedule */
    next = my_current_task->next;
    prev = my_current_task;
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {        
    	my_current_task = next; 
    	printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);  
    	/* switch to next process */
    	    	__asm__ volatile(	
            "mv %0,sp \n"
            "mv %1,ra \n"
            "ld sp,%2 \n"
            "ld ra,%3 \n"
            "ret"
        	: "=r" (prev->thread.sp),"=r" (prev->thread.ra)
        	: "m" (next->thread.sp), "m" ( next->thread.ra)
    	); 
    }  
    return;	
}
```

稍微解释一下汇编代码和C代码以及原理：（进程调度在纸上画个草图，我用LaTex和别的工具画最终的图）





mymain.c

```c
/*
 *  linux/mykernel/mymain.c
 *
 *  Kernel internal my_timer_handler
 *
 *  Copyright (C) 2023, 2023  WangRui
 *  
 */
#include <linux/types.h>
#include <linux/string.h>
#include <linux/ctype.h>
#include <linux/tty.h>
#include <linux/vmalloc.h>


#include "mypcb.h"

tPCB task[MAX_TASK_NUM];
tPCB * my_current_task = NULL;
volatile int my_need_sched = 0;

void my_process(void);


void __init my_start_kernel(void)
{
    int pid = 0;
    int i;
    /* Initialize process 0*/
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    task[pid].task_entry = task[pid].thread.ra = (unsigned long)my_process;
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    /*fork more process */
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
	    task[i].thread.sp = (unsigned long)(&task[i].stack[KERNEL_STACK_SIZE-1]);
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
	__asm__ volatile(
        "mv sp,%[sp]\n"
        "mv ra,%[ra]\n"
        "ret"
    	: 
    	: [ra] "r" (task[pid].thread.ra),[sp] "r" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
	);
} 

int i = 0;

void my_process(void)
{    
    while(1)
    {
        i++;
        if(i%10000000 == 0)
        {
            printk(KERN_NOTICE "this is process %d -\n",my_current_task->pid);
            if(my_need_sched == 1)
            {
                my_need_sched = 0;
        	    my_schedule();
        	}
        	printk(KERN_NOTICE "this is process %d +\n",my_current_task->pid);
        }     
    }
}
```

稍微解释一下汇编代码和C代码以及原理：



mypcb.h

```c
/*
 *  linux/mykernel/mypcb.h
 *
 *  Kernel internal PCB types
 *
 *  Copyright (C) 2023  WangRui
 *
 */

#define MAX_TASK_NUM        4
#define KERNEL_STACK_SIZE   1024*2
/* CPU-specific state of this task */
struct Thread {
    unsigned long		ra;
    unsigned long		sp;
};

typedef struct PCB{
    int pid;
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    unsigned long stack[KERNEL_STACK_SIZE];
    /* CPU-specific state of this task */
    struct Thread thread;
    unsigned long	task_entry;
    struct PCB *next;
}tPCB;

void my_schedule(void);

```

稍微解释一下汇编代码和C代码以及原理：



## 王瑞验证这部分实验的流程（可选）

### 编译RISC-V架构MyKernel内核

将上一节生成的linux内核目录替换VisionFive2中的linux内核目录后，使用以下命令进行编译。

```C
make -j$(nproc)
```

此次编译时间较长，编译成功后会在VisionFive2中自动创建work目录，所有的编译完成的文件都会在这个目录中生成。成功编译后，文件结构如图所示：

```C
work/
├── visionfive2_fw_payload.img
├── image.fit
├── initramfs.cpio.gz
├── u-boot-spl.bin.normal.out
├── linux/arch/riscv/boot
    ├── dts
    │   └── starfive
    │       ├── jh7110-visionfive-v2-ac108.dtb
    │       ├── jh7110-visionfive-v2.dtb
    │       ├── jh7110-visionfive-v2-wm8960.dtb
    │       ├── vf2-overlay
    │       │   └── vf2-overlay-uart3-i2c.dtbo
    └── Image.gz
```

### MyKernel内核移植VisionFive2开发板

## Qemu中Mykernel内核运行验证（可选）

#### Ubuntu安装和配置TFTP服务器

使用以下命令安装TFTP服务器：

```C
sudo apt-get install tftpd-hpa
```

查看 /etc/default/tftpd-hpa文件：

```bash
sudo apt install vim
sudo vim /etc/default/tftpd-hpa
```

文件内容如下图所示：

![image-20231029162404057](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231029162404057.png)

明确tftp服务器的文件路径在"/srv/tftp", 将上一节生成的work目录下的所有文件拷贝到/srv/tftp路径中。

#### VisionFive2端配置TFTP服务器

按照《使用Minicom连接VisionFive2》节中的步骤，连接VisionFive2和Ubuntu主机，待开发板成功引导Uboot后，使用下面的步骤进行配置。

1. 配置TFTP服务

   使用网线连接VisionFive2开发板后，通过路由器网管查看Ubuntu和VisionFive2开发板的IP地址。并使用以下的命令设置TFTP服务器的

   ```shell
   setenv 192.168.xxx.xxx; setenv serverip 192.168.xxx.xxx;
   ```