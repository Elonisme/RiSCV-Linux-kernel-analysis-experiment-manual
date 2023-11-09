# Linux 内核学习总结

Linux内核是一个开源、免费的操作系统内核，它是整个Linux操作系统的核心部分。但是由于Linux内核过于复杂，不易深入学习。由人民邮电出版社和中国工信出版社集团联合出版的《庖丁解牛Linux操作系统分析》一书，是我的恩师娄嘉鹏老师和孟宁老师共同编写的书籍并作为中科大软工学院和电科院研究生部的教材。自出版以来已获奖无数，该书内容翔实，通俗易懂，如今已在操作系统类图书排行上排名第六。

<img src="https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/mmexport1699537571114-removebg-preview.png" alt="《庖丁解牛Linux操作系统分析》" style="zoom: 50%;" />

本文集记录了该书中的八个实验，但与教材不同的是，本文记录的八个实验均是在RISC-V架构上的机器上实现的。RISC-V（发音为"risk-five"）是一种基于精简指令集计算机（RISC）原则的开放架构，被视为将来最有潜力与X86架构和Arm架构相竞争的新指令集架构。在本文集中所记录的实验中，除了需要使用GDB调试的部分使用了qemu进行模拟RISC-V架构的裸机之外，均使用了来自赛昉科技研发的全球首发的新一代RISC-V架构的开发板——Visionfive 2 实现的。

本文集从内容上来说，记录了比较详细的过程，使得读者能够尽可能容易的复现出所有实验。通过阅读和复现本文集的各个实验，你除了将学习到《庖丁解牛Linux操作系统分析》中记录的关于Linux 内核的知识点，还能学习到如何给RISC-V架构的开发板烧录GNU/LInux操作系统的发行版Debain操作系统；如何使用ssh和串口通讯连接RISC-V架构的开发板；OpenSBI和Uboot是何物; 一个简易的Linux操作系统将如何制作；如何使用TFTP协议给只带有Uboot的开发板传输文件；如何在Uboot上引导Linux内核和根文件系统；如何在X86平台上交叉编译RISC-V Linux内核；如何制作根文件系统；MenuOS如何在RISC-V架构上实现；RISC Linux内核是如何运行的等等。

本文集由我和[ZyLqb](https://github.com/ZyLqb)共同完成，由于我们两个现在还都是学生，因此本文集中难免会出现各种错误，请各位读者批评指正，欢迎提交Pr。

## 目录:

1. [反汇编一个简单的 C 程序](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab1反汇编一个简单的 C 程序.md#反汇编一个简单的-c-程序)
2. [完成一个简单的时间片轮转多道程序内核代码](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab2完成一个简单的时间片轮转多道程序内核代码.md#完成一个简单的时间片轮转多道程序内核代码)
3. [跟踪分析 Linux 内核的启动过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab3跟踪分析 Linux 内核的启动过程.md#跟踪分析-linux-内核的启动过程)
4. [使用库函数API和C代码中嵌入汇编代码两种方式使用同一个系统调用](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab4使用库函数 API 和 C 代码中嵌入汇编代码两种方式使用同一个系统调用.md#使用库函数api和c代码中嵌入汇编代码两种方式使用同一个系统调用)
5. [分析 system_call 中断处理过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab5分析 system_call 中断处理过程.md#分析-system_call-中断处理过程)
6. [分析 Linux 内核创建一个新进程的过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab6分析 Linux 内核创建一个新进程的过程.md#分析-linux-内核创建一个新进程的过程)
7. [Linux 内核如何装载和启动一个可执行程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab7Linux 内核如何装载和启动一个可执行程.md#linux-内核如何装载和启动一个可执行程)
8. [理解进程调度时机跟踪分析进程调度与进程切换的过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab8理解进程调度时机跟踪分析进程调度与进程切换的过程.md#理解进程调度时机跟踪分析进程调度与进程切换的过程)



