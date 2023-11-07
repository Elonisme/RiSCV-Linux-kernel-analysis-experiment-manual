# 使用库函数API和C代码中嵌入汇编代码两种方式使用同一个系统调用

## 使用SSH连接starfive visionfive 2

在第一个实验中，我们配置好了使用 `ssh` 连接 `starfive visionfive 2` 开发板的流程。在连接好网线后，通过使用浏览器登录网管地址查看开发板的ip地址后，使用以下命令连接 `starfive visionfive 2` 开发板。

```bash
ssh root@192.168.xxx.xxx
```

注意：`192.168.xxx.xxx` Ip地址需要使用自己查阅的Ip地址。

回车后，输入密码: `starfive`。进入 `starfive visionfive 2` 开发板中的shell。如下图所示：

![image-20231105133005556](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105133005556.png)

## 使用man查看write函数

使用以下命令查看write函数的具体用法

```bash
man 2 write
```

具体用法如图所示：

![image-20231105133327813](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105133327813.png)

write函数有三个参数。第0个参数，输出位置；第1个参数，指向输出内容的指针；第2个参数，输出内容的长度。

## C语言调用write函数

接下来，我们将在shell中vim编写`local_write.c`程序，通过使用C语言调用write函数。

首先我们需要下载 `vim`, 使用以下命令下载安装 `vim` 。

```bash
sudo apt install vim
```

安装好后，使用以下命令创建 `local_write.c` 文件。

```bash
vim loacl_write.c
```

将下面的代码复制到 `local_write.c` 中：

```c
#include <stdio.h>
#include <unistd.h>
int main(void){
	char s[]="hello, world.\n";
	write(1,s,13);
	return 0;
}
```

使用以下命令进行编译和运行：

```bash
gcc -o loacl_write -c local_wirte.c
./local_write
```

显示以下信息说明运行成功：

![image-20231105134051213](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105134051213.png)

## RISC-V内联汇编调用write函数

接下来，我们将使用RISC-V内联汇编嵌入C语言代码中，调用write函数。

使用以下命令创建 `asm_write.c` 文件。

```bash
vim asm_write.c
```

将下面的代码复制到 `asm_write.c` 中：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
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

使用以下命令进行编译和运行：

```bash
gcc -o asm_write -c asm_write.c
./asm_write
```

显示以下信息说明运行成功：

![image-20231105134323745](https://ellog.oss-cn-beijing.aliyuncs.com/ossimgs/image-20231105134323745.png)

### 内联汇编解释（王瑞）
