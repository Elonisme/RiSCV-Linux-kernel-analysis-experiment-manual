# Linux 内核学习总结

Linux内核是一个开源、免费的操作系统内核，它是整个Linux操作系统的核心部分。但是由于Linux内核过于复杂，不易深入学习。由人民邮电出版社和中国工信出版社集团联合出版的《庖丁解牛Linux操作系统分析》一书，是我的恩师娄嘉鹏老师和孟宁老师共同编写的书籍并作为中科大软工学院和电科院研究生部的教材。自出版以来已获奖无数，该书内容翔实，通俗易懂，如今已在操作系统类图书排行上排名第六。

<img src="https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/mmexport1699537571114-removebg-preview.png" alt="《庖丁解牛Linux操作系统分析》" style="zoom: 50%;" />

本文集记录了该书中的八个实验，但与教材不同的是，本文记录的八个实验均是在RISC-V架构上的机器上实现的。RISC-V（发音为"risk-five"）是一种基于精简指令集计算机（RISC）原则的开放架构，被视为将来最有潜力与X86架构和Arm架构相竞争的新指令集架构。在本文集中所记录的实验中，除了需要使用GDB调试的部分使用了qemu进行模拟RISC-V架构的裸机之外，均使用了来自赛昉科技研发的全球首发的新一代RISC-V架构的开发板——Visionfive 2 实现的。

本文集从内容上来说，记录了比较详细的过程，使得读者能够尽可能容易的复现出所有实验。通过阅读和复现本文集的各个实验，你除了将学习到《庖丁解牛Linux操作系统分析》中记录的关于Linux 内核的知识点，还能学习到如何给RISC-V架构的开发板烧录GNU/LInux操作系统的发行版Debain操作系统；如何使用ssh和串口通讯连接RISC-V架构的开发板；OpenSBI和Uboot是何物; 一个简易的Linux操作系统将如何制作；如何使用TFTP协议给只带有Uboot的开发板传输文件；如何在Uboot上引导Linux内核和根文件系统；如何在X86平台上交叉编译RISC-V Linux内核；如何制作根文件系统；MenuOS如何在RISC-V架构上实现；RISC Linux内核是如何运行的等等。

本文集由我和[ZyLqb](https://github.com/ZyLqb)共同完成，由于我们两个现在还都是学生，因此本文集中难免会出现各种错误，请各位读者批评指正，欢迎提交Pr。

## 目录:

1. [反汇编一个简单的 C 程序](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab1%E5%8F%8D%E6%B1%87%E7%BC%96%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%20C%20%E7%A8%8B%E5%BA%8F.md)
2. [完成一个简单的时间片轮转多道程序内核代码](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab2%E5%AE%8C%E6%88%90%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E6%97%B6%E9%97%B4%E7%89%87%E8%BD%AE%E8%BD%AC%E5%A4%9A%E9%81%93%E7%A8%8B%E5%BA%8F%E5%86%85%E6%A0%B8%E4%BB%A3%E7%A0%81.md)
3. [跟踪分析 Linux 内核的启动过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab3%E8%B7%9F%E8%B8%AA%E5%88%86%E6%9E%90%20Linux%20%E5%86%85%E6%A0%B8%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.md)
4. [使用库函数API和C代码中嵌入汇编代码两种方式使用同一个系统调用](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab4%E4%BD%BF%E7%94%A8%E5%BA%93%E5%87%BD%E6%95%B0%20API%20%E5%92%8C%20C%20%E4%BB%A3%E7%A0%81%E4%B8%AD%E5%B5%8C%E5%85%A5%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E4%B8%A4%E7%A7%8D%E6%96%B9%E5%BC%8F%E4%BD%BF%E7%94%A8%E5%90%8C%E4%B8%80%E4%B8%AA%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8.md)
5. [分析 system_call 中断处理过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab5%E5%88%86%E6%9E%90%20system_call%20%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86%E8%BF%87%E7%A8%8B.md)
6. [分析 Linux 内核创建一个新进程的过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab6%E5%88%86%E6%9E%90%20Linux%20%E5%86%85%E6%A0%B8%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E6%96%B0%E8%BF%9B%E7%A8%8B%E7%9A%84%E8%BF%87%E7%A8%8B.md)
7. [Linux 内核如何装载和启动一个可执行程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab7Linux%20%E5%86%85%E6%A0%B8%E5%A6%82%E4%BD%95%E8%A3%85%E8%BD%BD%E5%92%8C%E5%90%AF%E5%8A%A8%E4%B8%80%E4%B8%AA%E5%8F%AF%E6%89%A7%E8%A1%8C%E7%A8%8B.md)
8. [理解进程调度时机跟踪分析进程调度与进程切换的过程](https://github.com/Elonisme/RiSCV-Linux/blob/master/Lab8%E7%90%86%E8%A7%A3%E8%BF%9B%E7%A8%8B%E8%B0%83%E5%BA%A6%E6%97%B6%E6%9C%BA%E8%B7%9F%E8%B8%AA%E5%88%86%E6%9E%90%E8%BF%9B%E7%A8%8B%E8%B0%83%E5%BA%A6%E4%B8%8E%E8%BF%9B%E7%A8%8B%E5%88%87%E6%8D%A2%E7%9A%84%E8%BF%87%E7%A8%8B.md)

