
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1. 构建initramfs根文件系统](#1-构建initramfs根文件系统)
* [2. 编译调试版内核](#2-编译调试版内核)
* [3. 调试](#3-调试)
	* [3.2 第二种选择（对应1.2）](#32-第二种选择对应12)
* [4. 获取当前进程](#4-获取当前进程)
* [5. 参考](#5-参考)

<!-- /code_chunk_output -->


https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/

奔跑吧Linux内核/下内容

1. 编译调试版内核
2. 构建initramfs根文件系统
3. 调试
4. 获取当前进程
5. 参考

排查Linux内核Bug，研究内核机制，除了查看资料阅读源码，还可通过调试器，动态分析内核执行流程。

QEMU模拟器原生支持GDB调试器，这样可以很方便地使用GDB的强大功能对操作系统进行调试，如设置断点；单步执行；查看调用栈、查看寄存器、查看内存、查看变量；修改变量改变执行流程等。

用qemu+GDB来调试内核和ko，当然我们需要准备如下：

- 带调试信息的内核vmlinux
- 一个压缩的内核vmlinuz或者bzImage
- 一份裁剪过的文件系统initrd

## 1. 构建initramfs根文件系统

Linux系统启动阶段，boot loader加载完**内核文件vmlinuz后**，内核**紧接着**需要挂载磁盘根文件系统，但如果此时内核没有相应驱动，无法识别磁盘，就需要先加载驱动，而驱动又位于/lib/modules，得挂载根文件系统才能读取，这就陷入了一个两难境地，系统无法顺利启动。于是有了**initramfs根文件系统**，其中包含必要的设备驱动和工具，boot loader加载initramfs到内存中，内核会将其挂载到根目录/,然后**运行/init脚本**，挂载真正的磁盘根文件系统。

这里借助BusyBox构建极简initramfs，提供基本的用户态可执行程序。

编译BusyBox，配置CONFIG\_STATIC参数，编译静态版BusyBox，编译好的可执行文件busybox不依赖动态链接库，可以独立运行，方便构建initramfs。

```
$ cd busybox-1.28.0
$ make menuconfig
```

```
BusyBox Settings  --->
    Build Options  --->
        [*] Build BusyBox as a static binary (no shared libs)
```

Don't use /usr也一定要选,否则make install后busybox将安装在原系统的/usr下,这将覆盖掉系统原有的命令。选择这个选项后,make install后会在busybox目录下生成一个叫_install的目录,里面有busybox和指向它的链接.

```
BusyBox Settings  --->
    General Configuration  --->
        [*] Don't use /usr
```

```
$ make -j 20
$ make install
```

会安装在\_install目录:

```
$ ls _install 
bin  linuxrc  sbin  usr
```

把\_install 目录拷贝到 linux-4.0 目录下。进入\_install 目录，先创建 etc、dev 等目录。

```
$ mkdir etc
$ mkdir dev
$ mkdir mnt
$ mkdir -p etc/init.d/
```

在\_install/etc/init.d/目录下新创建一个叫 rcS 的文件，并且写入如下内容：

```
#!/bin/sh
export PATH=/sbin:/usr/sbin:/bin:/usr/bin

mkdir -p /proc
mkdir -p /tmp
mkdir -p /sys
mkdir -p /mnt

/bin/mount -a
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

需要给执行权限

```
chmod a+x etc/init.d/rcS
```

在\_install/etc 目录新创建一个叫 fstab 的文件，并写入如下内容。

```
proc /proc proc defaults 0 0
tmpfs /tmp tmpfs defaults 0 0
sysfs /sys sysfs defaults 0 0
tmpfs /dev tmpfs defaults 0 0
debugfs /sys/kernel/debug debugfs defaults 0 0
```

在\_install/etc 目录新创建一个叫 inittab 的文件，并写入如下内容。

```
::sysinit:/etc/init.d/rcS
::respawn:-/bin/sh
::askfirst:-/bin/sh
::ctrlaltdel:/bin/umount -a -r
```

在_install/dev 目录下创建如下设备节点，需要 root 权限。

```
$ cd _install/dev/
$ sudo mknod console c 5 1
$ sudo mknod null c 1 3
```

当然也可以使用既有的initramfs，或者将其进行裁剪（https://blog.csdn.net/weijitao/article/details/79477792）

## 2. 编译调试版内核

对内核进行调试需要解析符号信息，所以得编译一个调试版内核。

```
$ cd linux-4.14
$ make menuconfig
```

这里需要开启内核参数CONFIG\_DEBUG\_INFO和CONFIG\_GDB\_SCRIPTS。GDB提供了Python接口来扩展功能，内核基于Python接口实现了一系列辅助脚本，简化内核调试，开启CONFIG\_GDB\_SCRIPTS参数就可以使用了。

配置 initramfs，在 initramfs source file 中填入\_install。另外需要把 Default kernel command string 清空。

取消 Processor type and features \-\> Build a relocatable kernel 
取消后 Build a relocatable kernel 的子项 Randomize the address of the kernel image (KASLR) 也会一并被取消

```
Kernel hacking  ---> 
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
            [*] Generate dwarf4 debuginfo
        [*] Provide GDB scripts for kernel debugging

Processor type and features  ---> 
    -*- Build a relocatable kernel
        [ ] Randomize the address of the kernel image (KASLR)

General setup --->
    [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
        (_install) Initramfs source file(s)

Boot options -->
    ()Default kernel command string
```

```
$ make -j 20
```

编译后，bzImage这个是被压缩了的，不带调试信息的内核，供qemu虚拟机使用（arch/x86/boot/bzImage），vmlinux(当前目录)里面带了调试信息，没有压缩，供gdb使用。

当编译结束后，可以将vmlinux和bzImage文件copy到一个干净的目录下。


## 3. 调试

qemu 是一款虚拟机，可以模拟x86 & arm 等等硬件平台<似乎可模拟的硬件平台很多...>，而qemu 也内嵌了一个 gdbserver。这个gdbserver于是就可以和gdb构成一个远程合作伙伴，通过ip:port 网络方式或者是通过串口/dev/ttyS\*来进行工作，一个在这头，一个在那头。

### 3.2 第二种选择（对应1.2）

start\_kernel脚本：

```
#!/usr/bin/bash
qemu-system-x86_64 -M pc -smp 2 -m 1024 \
    -kernel arch/x86/boot/bzImage \
    -append "rdinit=/linuxrc loglevel=7" -S -s
```

启动gdb脚本：

```
#!/bin/bash

gdb ./vmlinux -ex "target remote localhost:1234"    \
              -ex "break start_kernel"              \
              -ex "continue"                        \
              -ex "disconnect"                      \
              -ex "set architecture i386:x86-64:intel" \
              -ex "target remote localhost:1234"
```

## 4. 获取当前进程

《深入理解Linux内核》第三版第三章–进程，讲到内核采用了一种精妙的设计来获取当前进程。

![config](images/1.jpg)

Linux把跟一个进程相关的thread\_info和内核栈stack放在了同一内存区域，内核通过esp寄存器获得当前CPU上运行进程的内核栈栈底地址，该地址正好是thread\_info地址，由于进程描述符指针task字段在thread\_info结构体中偏移量为0，进而获得task。相关汇编指令如下：

```
movl $0xffffe000, %ecx      /* 内核栈大小为8K，屏蔽低13位有效位。
andl $esp, %ecx
movl (%ecx), p
```

指令运行后，p就获得当前CPU上运行进程的描述符指针。

然而在调试器中调了下，发现**这种机制早已经被废弃掉了**。thread\_info结构体中只剩下一个字段flags，进程描述符字段task已经删除，无法通过thread\_info获取进程描述符了。

而且进程的thread\_info也不再位于进程内核栈底了，而是放在了进程描述符task\_struct结构体中，见提交sched/core: Allow putting thread\_info into task\_struct和x86: Move thread\_info into task\_struct，这样也无法通过esp寄存器获取thread\_info地址了。

```
(gdb) p $lx_current().thread_info
$5 = {flags = 2147483648}
```

这样做是从安全角度考虑的，一方面可以防止esp寄存器泄露后进而泄露进程描述符指针，二是防止内核栈溢出覆盖thread_info。

Linux内核从2.6引入了Per-CPU变量，获取当前指针也是通过Per-CPU变量实现的。

```
(gdb) p $lx_current().pid
$50 = 77
(gdb) p $lx_per_cpu("current_task").pid
$52 = 77
```

## 5. 参考

- [Tips for Linux Kernel Development](http://eisen.io/slides/jeyu_tips_for_kernel_dev_cmps107_2017.pdf)
- [How to Build A Custom Linux Kernel For Qemu](http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html)
- [Linux Kernel System Debugging](https://blog.0x972.info/?d=2014/11/27/18/45/48-linux-kernel-system-debugging-part-1-system-setup)
- [Debugging kernel and modules via gdb](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html)
- [BusyBox simplifies embedded Linux systems](https://www.ibm.com/developerworks/library/l-busybox/index.html)
- [Custom Initramfs](https://wiki.gentoo.org/wiki/Custom_Initramfs)
- [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
- [Linux kernel debugging with GDB: getting a task running on a CPU](http://slavaim.blogspot.com/2017/09/linux-kernel-debugging-with-gdb-getting.html)