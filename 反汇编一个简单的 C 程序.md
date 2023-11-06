#  反汇编一个简单的 C 程序

## 将OS烧录到Micro-SD卡上

现在我们需要将Debian（Linux发行版）烧录到Micro-SD卡上，以便于它可以在昉·星光 2上运行

1. 使用Micro-SD卡读卡器或笔记本电脑上的内置读卡器，将Micro-SD卡连接至计算机。
2. 点击[此链接](https://debian.starfivetech.com/)下载最新Debian镜像。
3. 解压.bz2文件。
4. 访问[此链接](https://www.balena.io/etcher/)下载BalenaEtcher。我们将使用BalenaEtcher将Debian镜像烧录到Micro-SD卡上。

![image-20231015121217662](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231015121217662.png)

## ssh连接visionfive 2

1. 通过HDMI使用Xfce桌面环境登录

   1. **Username**: root
   2. **Password**: starfive

2. 允许root用户通过ssh登录

   ``` shell
   echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
   ```

3. 重启ssh服务器

   ```shell
   /etc/init.d/ssh restart
   ```

4. 登录到路由器找到昉·星光 2的IP地址

5. 通过ssh登录开发版

   ```shell
   ssh root@192.168.1.xxx
   ```

6. 输入密码完成登录

## 安装gcc和vim

``` shell
apt update
apt install gcc vim
```

完成登录

## 反编译c语言代码

使用vim创建main.c

``` shell
vim main.c
```
复制以下代码：

```c
// main.c
int g(int x)
{
    return x + 3;
}
    
int f(int x)
{
    return g(x);
}
    
int main(void)
{
    return f(8) + 1;
}
```

使用gcc命令反编译c语言到risc-v汇编
```shell
gcc -S -o main.s main.c
```
得到main.s后，使用vim打开
```assembly
	.file	"main.c"
	.option pic
	.text
	.align	1														
	.globl	g
	.type	g, @function
g:
	addi	sp,sp,-32
	sd	s0,24(sp)
	addi	s0,sp,32
	mv	a5,a0
	sw	a5,-20(s0)
	lw	a5,-20(s0)
	addiw	a5,a5,3
	sext.w	a5,a5
	mv	a0,a5
	ld	s0,24(sp)
	addi	sp,sp,32
	jr	ra
	.size	g, .-g
	.align	1
	.globl	f
	.type	f, @function
f:
	addi	sp,sp,-32
	sd	ra,24(sp)
	sd	s0,16(sp)
	addi	s0,sp,32
	mv	a5,a0
	sw	a5,-20(s0)
	lw	a5,-20(s0)
	mv	a0,a5
	call	g
	mv	a5,a0
	mv	a0,a5
	ld	ra,24(sp)
	ld	s0,16(sp)
	addi	sp,sp,32
	jr	ra
	.size	f, .-f
	.align	1
	.globl	main
	.type	main, @function
main:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	li	a0,8
	call	f
	mv	a5,a0
	addiw	a5,a5,1
	sext.w	a5,a5
	mv	a0,a5
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
	.size	main, .-main
	.ident	"GCC: (Debian 11.3.0-3) 11.3.0"
	.section	.note.GNU-stack,"",@progbits
```

在vim的命令模式下，使用 :g/.*\./d 删除带有的. 的行


```assembly
g:
	addi	sp,sp,-32
	sd	s0,24(sp)
	addi	s0,sp,32
	mv	a5,a0
	sw	a5,-20(s0)
	lw	a5,-20(s0)
	addiw	a5,a5,3
	mv	a0,a5
	ld	s0,24(sp)
	addi	sp,sp,32
	jr	ra
f:
	addi	sp,sp,-32
	sd	ra,24(sp)
	sd	s0,16(sp)
	addi	s0,sp,32
	mv	a5,a0
	sw	a5,-20(s0)
	lw	a5,-20(s0)
	mv	a0,a5
	call	g
	mv	a5,a0
	mv	a0,a5
	ld	ra,24(sp)
	ld	s0,16(sp)
	addi	sp,sp,32
	jr	ra
main:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	li	a0,8
	call	f
	mv	a5,a0
	addiw	a5,a5,1
	mv	a0,a5
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
```