# Linux 内核如何装载和启动一个可执行程

## 实验要求： 

1. 理解编译链接的过程和 ELF 可执行文件格式

2. 编程使用 exec库函数加载一个可执行文件，动态链接分为可执行程序装载时动态链接和运行时动态链接，编程练习动态链接库的这两种使用方式
3. 使用 gdb 跟踪分析一个 execve 系统调用内核处理函数 sys_execve ，验证您对 Linux 系统加载可执行程序所需处理过程的理解。特别关注新的可执行程序是从哪里开始执行的？为什么 execve 系统调用返回后新的可执行程序能顺利执行？对于静态链接的可执行程序和动态链接的可执行程序 execve 系统调用返回时会有什么不同？

## 程序的编译过程

程序从源代码到可执行文件的编译步骤分为4步：

```mermaid
graph TD;
预处理-->编译;
编译-->汇编;
汇编-->链接;
```

在预处理过程中，主要是完成删除和展开以及处理命令这三种操作，最后将文本保存到后缀为 `.i` 文件中。以预处理 `hello.c` 文件为例。

下面是 `hello.c` 文件的内容：

```c
#include <stdio.h>

void main(){
	printf("hello world!\n");
}
```

使用ssh 连接好 `visionfive 2` 开发板后，使用vim创建 `hello.c` 文件。

然后执行以下的预处理命令得到 `hello.i` 文件:

```bash
gcc -E hello.c -o hello.i
```

使用 vim 打开 `hello.i` 文件后内容较多，但可以看出有几大特点：

首先是 `#include` 命令全部替换成包含文件的路径，如下图所示：

![image-20231108152215370](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231108152215370.png)

同时添加了行号和文件名标示：

![image-20231108152332267](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231108152332267.png)

其次是删除了所有的注释。

编译是在预处理的基础上，gcc首先检查代码的规范性和语法错误，检查无误后的翻译为汇编语言。

使用以下的命令将预处理的中的代码生成为汇编语言代码：

```c
gcc -S hello.i -o hello.s
```

以下的在 `visionfive 2` 中编译完的RISCV架构的汇编代码：

```c
	.file	"hello.c"
	.option pic
	.attribute arch, "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0"
	.attribute unaligned_access, 0
	.attribute stack_align, 16
	.text
	.section	.rodata
	.align	3
.LC0:
	.string	"Hello world!"
	.text
	.align	1
	.globl	main
	.type	main, @function
main:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	lla	a0,.LC0
	call	puts@plt
	nop
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
	.size	main, .-main
	.ident	"GCC: (Debian 12.2.0-10) 12.2.0"
	.section	.note.GNU-stack,"",@progbits
```

链接就是将各个代码和数据收集起来组合成一个单一文件的过程。以下的命令将生成RISC-V Linux下的可执行文件：

```c
gcc hello.o -o hello -static
```

以下是节表头：

```
There are 28 section headers, starting at offset 0x7a490:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.bu[...] NOTE             00000000000101c8  000001c8
       0000000000000024  0000000000000000   A       0     0     4
  [ 2] .note.ABI-tag     NOTE             00000000000101ec  000001ec
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .rela.dyn         RELA             0000000000010210  00000210
       0000000000000210  0000000000000018   A      25     0     8
  [ 4] .text             PROGBITS         0000000000010420  00000420
       00000000000412b2  0000000000000000  AX       0     0     4
  [ 5] __libc_freeres_fn PROGBITS         00000000000516d2  000416d2
       0000000000000814  0000000000000000  AX       0     0     2
  [ 6] .rodata           PROGBITS         0000000000051ef0  00041ef0
       000000000001b4a4  0000000000000000   A       0     0     16
```

## 编程使用 exec库函数加载一个可执行文件

在  `visionfive 2` 开发板上启动debain后，使用以下的命令使用man命令查看execve函数信息：

```bash
man execve
```

execve函数信息如下：

![image-20231108204744069](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231108204744069.png)

现在编写一个叫作 `local_exec.c` 的文件，使用 `exec` 系统调用加载之前编译产生的 `hello` 可执行文件。下面的是 `local_exec.c` 的内容：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
	int pid;
	/* fork another process */
	pid = fork();
	if (pid < 0) 
	{ 
		/* error occurred */
		fprintf(stderr,"Fork Failed!");
		exit(-1);
	} 
	else if (pid == 0) 
	{
		/*	 child process 	*/
    	printf("This is Child Process!\n");
		execlp("/hello","hello",NULL);
	} 
	else 
	{ 	
		/* 	parent process	 */
    	printf("This is Parent Process!\n");
		/* parent will wait for the child to complete*/
		wait(NULL);
		printf("Child Complete!\n");
	}
}
```

在 `execl` 函数中，第一个参数是可执行文件的路径，第二个参数是程序的名称，最后一个参数必须是 `(char *)0`。

使用以下命令进行编译：

```c
gcc -o local_exec local_exec.c
```

运行结果如下：

![image-20231109161948754](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109161948754.png)

## 动态链接

 动态链接是一种在程序运行时将代码和数据与程序链接的技术，而不是在编译时将它们链接到可执行文件中。它提供了灵活性和共享性，允许多个程序共享和重复使用相同的库代码。动态链接主要分为两种方式：可执行程序装载时动态链接和运行时动态链接。相比于静态链接，动态链接可以多个程序共享同一个段代码，而不需要多个副本。但是动态链接对库的依赖程度高。

可执行程序装载时动态链接是指动态链接发生在可执行程序被加载到内存时查找和链接动态链接库。运行时动态链接是指程序在运行时显式加载并链接动态链接库，然后调用库中的函数。

可执行程序装载时动态链接通常更加高效，因为它减少了运行时的开销，而运行时动态链接提供了更大的灵活性，允许动态加载和卸载库，以便在运行时进行插件或模块式的扩展。

gcc默认编译就是动态链接，使用以下命令动态链接 `hello` 可执行文件：

```c
gcc hello.o -o hello.dynamic
```

使用 ` ls -l` 命令可以发现在RISC-V架构下动态链接的可执行文件比静态链接的可执行文件小58倍，而书上比较x86的比值是100倍：

![image-20231108210901120](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231108210901120.png)

以下是用于实践动态链接的代码：

首先是可执行程序装载时动态链接代码，我们需要创建两个文件，分别是 `shlibexample.h` 和 `shlibexample.c` 文件。

`shlibexample.h` 源代码如下：

```c
#ifndef _SH_LIB_EXAMPLE_H_
#define _SH_LIB_EXAMPLE_H_

#define SUCCESS 0
#define FAILURE (-1)

#ifdef __cplusplus
extern "C"{
	#endif
	int SharedLibApi();
	#ifdef __cplusplus
}
#endif
#endif
```

`shlibexample.c ` 源代码如下：

```c
#include <stdio.h>
#include "shlibexample.h"

int SharedLibApi(){
	printf("This is a shared library!\n");
	return SUCCESS;
}
```

保存好后文件后，在`visionfive 2` 开发板使用以下命令，创建 `libshlibexample.so` 文件:

```c
gcc -shared shlibexample.c -o libshlibexample.so
```

实现运行时动态链接，我们也需要创建两个文件，分别是 `dllibexample.h` 和 `dllibexample.c` 文件。

`dllibexample.h` 源代码如下：

```c
#ifndef _DL_LIB_EXAMPLE_H_
#define _DL_LIB_EXAMPLE_H_

#ifdef __cplusplus
extern "C"{
    #endif
    
	int DynamicalLoadingLibAPi();
	
	#ifdef __cplusplus
}
#endif
#endif
```

`dllibexample.c` 源代码如下：

```c
#include <stdio.h>
#include "dllibexample.h"

#define SUCCESS 0
#define FAILURE (-1)

int DynamicalLoadingLibAPi(){
	printf("This is a Dynamical Loading library!\n");
	return SUCCESS;
}
```

同样保存好后文件后，在`visionfive 2` 开发板使用以下命令，创建 `libdlibexample.so` 文件:

```c
gcc -shared dllibexample.c -o libdlibexample.so
```

当分别生成好 `libshlibexample.so` 文件和 `libdlibexample.so` 文件后，我们需要分别调用这两个文件。在`visionfive 2` 开发板中创建一个名为 `main.c` 的文件，`main.c` 的内容如下：

```
#include <stdio.h>
#include "shlibexample.h"
#include <dlfcn.h>

int main(){
	printf("Calling SharedLibApi() function of libshlibexample.so!\n");
	SharedLibApi();
	
	void * handle = dlopen("libdlibexample.so", RTLD_NOW);
	if(handle == NULL){
		printf("Open Lib libdllibexample.so Error:%s\n", dlerror());
		return FAILURE;
	}
	int (*func)(void);
	char *error;
	func = dlsym(handle ,"DynamicalLoadingLibAPi");
	if((error = dlerror()) != NULL){
		printf("DynamicalLoadingLibAPi() not found:%s\n", error);
		return FAILURE;
	}
	printf("Calling DynamicalLoadingLibApi() function of libdlibexample.so!\n");
	func();
	dlclose(handle);
	return SUCCESS;
}
```

编辑好保存后，使用以下命令编译成 `main` 可执行文件：

```c
gcc main.c -o main -L/root/Code/ -lshlibexample -ldl
```

然后将之前生成的 `shlibexample.so` 文件拷贝到 `/usr/local/lib` 中否则程序无法找到 `shlibexample.so`文件。完成以上工作后，使用以下命令，即可实现可执行程序装载时动态链接和运行时动态链接:

```bash
./main	
```

成功运行后，会显示以下信息：

![image-20231108220153906](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231108220153906.png)

## gdb 跟踪分析一个 execve 系统调用

进入MenuOS目录中的menuos目录后，编辑test.c, 在其中加入以下代码：

首先添加以下的头文件

```c
#include <unistd.h>
```

然后添加 `Exec` 函数：

```c
int Exec(int argc, char *argv[])
{
	int pid;
	/* fork another process */
	pid = fork();
	if (pid < 0)
	{
		/* error occurred */
		fprintf(stderr,"Fork Failed!");
		exit(-1);
	}
	else if (pid == 0)
	{
		/*	 child process 	*/
    	printf("This is Child Process!\n");
		execlp("/hello","hello",NULL);
	}
	else
	{
		/* 	parent process	 */
    	printf("This is Parent Process!\n");
		/* parent will wait for the child to complete*/
		wait(NULL);
		printf("Child Complete!\n");
	}
}
```

最后在 `main` 函数中添加以下语句：

```c
MenuConfig("execve", "execve new process", Exec);
```

在menuos目录下使用以下语句进行编译：

```c
make
make rootfs
```

编译完成后，在两个终端上分别启动 `init-gdb.sh` 和 `start-gdb.sh` 脚本。脚本启动后，在启动 `start-gdb.sh` 的终端上输入 `target remote:1234` 连接qemu中的gdbserver。完成后，使用下述命令在gdb中为 `sys_execve` 设置断点：

```gdb
b sys_execve
```

打好断点后，使用 `c` 命令，加载MenuOS后输入 `exec` 开始调试 `exec`，如下图所示：

![image-20231109163507762](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109163507762.png)

首先跳转到 `SYSCALL_DEFINE3(execve, ....) `：

![image-20231109164518819](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109164518819.png)

函数体中有 `do_execve() `  ，然后使用 `b do_execve`  命令进行打断点：

![image-20231109165602046](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109165602046.png)

进入 `do_execve() ` 后调用 `do_execveat_common` 函数, 现在给`do_execveat_common` 函数打上断点：

![image-20231109165917824](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109165917824.png)

而大部分的工作基本上都是在 `do_execveat_common` 函数内完成的，`do_execveat_common` 函数执行完后会依次返回到 `SYSCALL_DEFINE3(execve, ....) ` 中完成调用, 完成调用后 MenuOS中的Shell将显示 `hello world!`:

![image-20231109165138037](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109165138037.png)

但此时，进程仍然没有结束，下一步将进入 `schedule` 函数中依次执行 `schedule` 函数中的内容:

![image-20231109164729388](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109164729388.png)

 `schedule` 函数结束后，将再次引发 `Shell` 程序引起的中断：

![image-20231109170505625](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231109170505625.png)

此时MenuOS中的Shell可以再次输入命令。
