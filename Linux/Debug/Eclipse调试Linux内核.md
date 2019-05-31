
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 两个环境(远程调试)](#1-两个环境远程调试)
	* [1.1 初始化环境配置](#11-初始化环境配置)
	* [1.2 启动远端gdbserver](#12-启动远端gdbserver)
	* [1.3 配置eclipse](#13-配置eclipse)

<!-- /code_chunk_output -->

使用QEMU \+ GDB \+ Eclipse调试Linux内核

有两种情况, 一种是内核虚拟机和eclipse在同一个环境, 另一种分属两个环境

# 1 两个环境(远程调试)

## 1.1 初始化环境配置

本地安装Eclipse for Cpp

本地checkout一份代码出来, 该代码和编译的调试用的kernel commit信息一致

本地安装gdb

远端的bzImage作为虚拟机启动镜像

本地vmlinux是带有调试信息的镜像, 与上面的对应

## 1.2 启动远端gdbserver

```
qemu-system-x86_64 -smp 2 -m 1024 -kernel arch/x86/boot/bzImage -nographic -append "rdinit=/linuxrc loglevel=8 console=ttyS0" -S -s
```

或

```
qemu-system-x86_64 -smp 2 -m 1024 -kernel arch/x86/boot/bzImage -append "rdinit=/linuxrc loglevel=8" -S -s -daemonize
```

## 1.3 配置eclipse

右上角"Debug", 选择"C/C\+\+"

![](./images/2019-05-31-12-51-02.png)

new → "Makefile Project with Existing Code", 这里代码目录选择上面说的与调试内核代码一致的目录, Toolchain选为None.

![](./images/2019-05-31-14-06-14.png)

配置debug选项, "Run" → "Debug Configurations", 选择"C/C\+\+ Attach to Application"双击, 新建一个配置, 起个名字

Main中

Project选择刚才创建的project, C/C\+\+ Application选择上面所说的vmlinux. 

![](./images/2019-05-31-14-37-13.png)

Debugger中

Debugger选择"gdbserver"

Main中, 如下

![](./images/2019-05-31-14-40-39.png)

connection中, 如下, 对端IP填写, 端口为1234

![](./images/2019-05-31-14-41-04.png)

点击右下的"debug"


