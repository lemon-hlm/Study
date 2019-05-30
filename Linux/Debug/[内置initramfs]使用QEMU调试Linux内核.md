
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 构建initramfs根文件系统](#1-构建initramfs根文件系统)
* [2 编译调试版内核](#2-编译调试版内核)
* [3 调试](#3-调试)
* [4 问题和解决方案](#4-问题和解决方案)
* [5. 参考](#5-参考)

<!-- /code_chunk_output -->

用qemu+GDB来调试内核和ko，当然我们需要准备如下：

- 带调试信息的内核vmlinux
- 一个压缩的内核vmlinuz或者bzImage
- 一份裁剪过的文件系统initrd

# 1 构建initramfs根文件系统

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

# 2 编译调试版内核

对内核进行调试需要解析符号信息，所以得编译一个调试版内核。

```
$ cd linux-4.14
$ make menuconfig
```

这里需要开启内核参数CONFIG\_DEBUG\_INFO和CONFIG\_GDB\_SCRIPTS。GDB提供了Python接口来扩展功能，内核基于Python接口实现了一系列辅助脚本，简化内核调试，开启CONFIG\_GDB\_SCRIPTS参数就可以使用了。

配置 initramfs，在 initramfs source file 中填入\_install。另外需要把 Default kernel command string 清空。

取消 Processor type and features \-\> Build a relocatable kernel: 因为内核启用这项特性之后，内核启动时会随机化内核的各个 section 的虚拟地址（VA），导致 gdb 断点设置在错误的虚拟地址上，内核执行时就不会触发这些断点。

取消后 Build a relocatable kernel 的子项 Randomize the address of the kernel image (KASLR) 也会一并被取消

Generate dwarf4 debuginfo: 方便 gdb 调试。可参考 dwarf4 格式。

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


# 3 调试

qemu 是一款虚拟机，可以模拟x86 & arm 等等硬件平台<似乎可模拟的硬件平台很多...>，而qemu 也内嵌了一个 gdbserver。这个gdbserver于是就可以和gdb构成一个远程合作伙伴，通过ip:port 网络方式或者是通过串口/dev/ttyS\*来进行工作，一个在这头，一个在那头。

start\_kernel脚本：

```
#!/usr/bin/bash
qemu-system-x86_64 -smp 2 -m 1024 -kernel arch/x86/boot/bzImage -nographic -append "rdinit=/linuxrc loglevel=8 console=ttyS0" -S -s
```

启动gdb脚本或参照下面命令

```
#!/bin/bash

gdb ./vmlinux -ex "target remote localhost:1234"    \
              -ex "break start_kernel"              \
              -ex "continue"                        \
              -ex "disconnect"                      \
              -ex "set architecture i386:x86-64:intel" \
              -ex "target remote localhost:1234"
```

```
# vmlinux 是编译内核时生成的调试文件，在内核源码的根目录中。
gdb vmlinux
# 进入 gdb 的交互模式后，首先执行
show arch
# 当前架构一般是: i386:x86-64

# 连接 qemu 进行调试：
target remote :1234

# 设置断点
# 如果上面qemu是使用 qemu-kvm 执行的内核的话，就需要使用 hbreak 来设置断点，否则断点无法生效。
# 但是我们使用的是 qemu-system-x86_64，所以可以直接使用 b 命令设置断点。
b start_kernel
# 执行内核
c
```

执行内核后，gdb 会出现一个错误：

```
Remote 'g' packet reply is too long: 后续一堆的十六进制数字
```

这是 gdb 的一个bug，可以通过以下方式规避：

```
# 断开 gdb 的连接
disconnect
# 重新设置 arch
# 此处设置和之前 show arch 的要不一样
# 之前是  i386:x86-64 于是改成  i386:x86-64:intel
set arch i386:x86-64:intel
```

设置完 arch 后，重新连接：

```
target remote :1234
```

连接上后就可以看到 gdb 正常的输出 start\_kernel 处的代码，然后按照 gdb 的调试指令进行内核调试即可。

直到最后完全启动, 到达根目录

```
/ # ls -l
total 0
drwxr-xr-x    2 0        0             5320 May 30 07:46 bin
drwxrwxrwt    6 0        0             2060 May 30 08:21 dev
drwxr-xr-x    3 0        0              100 May 30 07:46 etc
lrwxrwxrwx    1 0        0               11 May 30 07:46 linuxrc -> bin/busybox
drwxr-xr-x    2 0        0               40 May 30 07:46 mnt
dr-xr-xr-x   57 0        0                0 May 30 08:21 proc
drwxr-xr-x    2 0        0             2620 May 30 07:46 sbin
dr-xr-xr-x   13 0        0                0 May 30 08:21 sys
drwxrwxrwt    2 0        0               40 May 30 08:21 tmp
```

这就是initramfs中包含的内容

# 4 问题和解决方案

1. 为什么要关闭 Build a relocatable kernel 

因为内核启用这项特性之后，内核启动时会随机化内核的各个 section 的虚拟地址（VA），导致 gdb 断点设置在错误的虚拟地址上，内核执行时就不会触发这些断点。

2. Generate dwarf4 debuginfo 有什么用 

方便 gdb 调试。可参考 dwarf4 格式。

3. Remote 'g' packet reply is too long 错误的原因 

这个错误是当目标程序执行时发生模式切换（real mode 16bit -> protected mode 32bit -> long mode 64bit）的时候，gdb server（此处就是 qemu）发送了不同长度的信息，gdb 无法正确的处理这种情况，所以直接就报错。 
此时需要断开连接并切换 gdb 的 arch （i386:x86-64 和 i386:x86-64:intel ），arch 变化后，gdb 会重新设置缓冲区，然后再连接上去就能正常调试。这个方法规避了一些麻烦，但是实际上有两种正规的解决方案： 
（1） 修改 gdb 的源码，使 gdb 支持这种长度变化（gdb 开发者似乎认为这个问题应该由 gdb server 解决）。 
（2） 修改 qemu 的 gdb server，始终发送 64bit 的消息（但是这种方式可能导致无法调试 real mode 的代码）。

4. 为什么最后内核执行出现了 Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) 

因为 qemu 没有加载 rootfs，所以内核最后挂 VFS 的时候会出错。可以用 busybox 构建一个文件系统镜像，然后 qemu 增加 -initrd 选项指向该文件系统镜像即可。

# 5. 参考

- [Tips for Linux Kernel Development](http://eisen.io/slides/jeyu_tips_for_kernel_dev_cmps107_2017.pdf)
- [How to Build A Custom Linux Kernel For Qemu](http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html)
- [Linux Kernel System Debugging](https://blog.0x972.info/?d=2014/11/27/18/45/48-linux-kernel-system-debugging-part-1-system-setup)
- [Debugging kernel and modules via gdb](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html)
- [BusyBox simplifies embedded Linux systems](https://www.ibm.com/developerworks/library/l-busybox/index.html)
- [Custom Initramfs](https://wiki.gentoo.org/wiki/Custom_Initramfs)
- [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
- [Linux kernel debugging with GDB: getting a task running on a CPU](http://slavaim.blogspot.com/2017/09/linux-kernel-debugging-with-gdb-getting.html)
- 使用 QEMU 和 GDB 调试 Linux 内核 v4.12: https://imkira.com/a21.html
- https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/
- QEMU+GDB调试内核: https://blog.csdn.net/weijitao/article/details/79477792