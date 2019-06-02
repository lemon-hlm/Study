
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 QEMU 中使用 BIOS 简介](#1-qemu-中使用-bios-简介)
	* [清单1 QEMU 源码树中的 BIOS 文件](#清单1-qemu-源码树中的-bios-文件)
	* [清单 2 QEMU 源码树以子模块方式保存的 BIOS 代码](#清单-2-qemu-源码树以子模块方式保存的-bios-代码)
	* [清单 3 QEMU 的 Makefile 中关于 BIOS 的拷贝操作](#清单-3-qemu-的-makefile-中关于-bios-的拷贝操作)
* [2 QEMU 加载 BIOS 过程分析](#2-qemu-加载-bios-过程分析)
	* [清单 4 1.7.0 版本 x86_64 QEMU 中支持的类型](#清单-4-170-版本-x86_64-qemu-中支持的类型)
	* [清单 5 QEMU 中 MemoryRegion 结构体](#清单-5-qemu-中-memoryregion-结构体)
	* [清单 6 old\_pc\_system\_rom\_init函数中将 BIOS 映射到物理内存空间的代码：](#清单-6-old_pc_system_rom_init函数中将-bios-映射到物理内存空间的代码)
* [3 小结](#3-小结)

<!-- /code_chunk_output -->

https://www.linuxidc.com/Linux/2014-12/110472.htm

QEMU 是一个广泛使用的开源计算机仿真器和虚拟机，它提供了虚拟机硬件的虚拟化功能，其使用的某些特定硬件的固件则由一些开源项目提供。本文将介绍 QEMU 代码中使用到的 BIOS，通过分析 QEMU 代码，讲解 BIOS 是如何加载到虚拟机的物理内存。

# 1 QEMU 中使用 BIOS 简介

BIOS 提供主板或者显卡的固件信息以及基本输入输出功能，QEMU使用的是一些开源的项目，如 Bochs、openBIOS等。

QEMU中使用到的BIOS以及固件一部分以二进制文件的形式保存在**源码树的pc-bios目录下**，pc-bios目录里包含了QEMU使用到的**固件**。

还有一些**BIOS以git源代码子模块**的形式**保存在QEMU的源码仓库**中，当编译QEMU程序的时候，也同时编译出这些BIOS或者固件的二进制文件。

QEMU支持多种启动方式，比如说efi、pxe 等，都包含在该目录下，这些都需要特定BIOS的支持。

## 清单1 QEMU 源码树中的 BIOS 文件

![config](images/1.png)

## 清单 2 QEMU 源码树以子模块方式保存的 BIOS 代码

```
$ cat .gitmodules
[submodule "roms/vgabios"]
        path = roms/vgabios
        url = git://git.qemu-project.org/vgabios.git/
[submodule "roms/seabios"]
        path = roms/seabios
        url = git://git.qemu-project.org/seabios.git/
[submodule "roms/SLOF"]
        path = roms/SLOF
        url = git://git.qemu-project.org/SLOF.git
[submodule "roms/ipxe"]
        path = roms/ipxe
        url = git://git.qemu-project.org/ipxe.git
[submodule "roms/openbios"]
        path = roms/openbios
        url = git://git.qemu-project.org/openbios.git
[submodule "roms/openhackware"]
        path = roms/openhackware
        url = git://git.qemu-project.org/openhackware.git
[submodule "roms/qemu-palcode"]
        path = roms/qemu-palcode
        url = git://github.com/rth7680/qemu-palcode.git
[submodule "roms/sgabios"]
        path = roms/sgabios
        url = git://git.qemu-project.org/sgabios.git
[submodule "dtc"]
        path = dtc
        url = git://git.qemu-project.org/dtc.git
[submodule "roms/u-boot"]
        path = roms/u-boot
        url = git://git.qemu-project.org/u-boot.git
[submodule "roms/skiboot"]
        path = roms/skiboot
        url = git://git.qemu.org/skiboot.git
[submodule "roms/QemuMacDrivers"]
        path = roms/QemuMacDrivers
        url = git://git.qemu.org/QemuMacDrivers.git
[submodule "ui/keycodemapdb"]
        path = ui/keycodemapdb
        url = git://git.qemu.org/keycodemapdb.git
[submodule "capstone"]
        path = capstone
        url = git://git.qemu.org/capstone.git
```

当我们从源代码编译 QEMU 时候，QEMU 的 Makefile 会将这些二进制文件拷贝到 QEMU 的数据文件目录中。

## 清单 3 QEMU 的 Makefile 中关于 BIOS 的拷贝操作

```
ifneq ($(BLOBS),)
    set -e; for x in $(BLOBS); do \
        $(INSTALL_DATA) $(SRC_PATH)/pc-bios/$$x "$(DESTDIR)$(qemu_datadir)"; \
    done
endif
```

# 2 QEMU 加载 BIOS 过程分析

当QEMU用户空间进程开始启动时，QEMU进程会根据所**传递的参数**以及当前**宿主机平台类型(host类型）**，自动加载适当的BIOS固件。QEMU进程启动初始阶段，会通过module\_call\_init函数调用qemu\_register\_machine注册**该平台支持的全部机器类型**，接着调用find\_default\_machine**选择一个默认的机型**进行初始化。 以QEMU代码（1.7.0）的x86_64平台为例，支持的机器类型有：

## 清单 4 1.7.0 版本 x86_64 QEMU 中支持的类型

```
pc-q35-1.7 pc-q35-1.6 pc-q35-1.5 pc-q35-1.4 pc-i440fx-1.7 pc-i440fx-1.6 pc-i440fx-1.5
pc-i440fx-1.4 pc-1.3 pc-1.2 pc-1.1 pc-1.0 pc-0.15 pc-0.14
pc-0.13    pc-0.12    pc-0.11    pc-0.10    isapc
```

代码中使用的默认机型为pc-i440fx-1.7，使用的 BIOS 文件为：

```
pc-bios/bios.bin
Default machine name : pc-i440fx-1.7
bios_name = bios.bin
```

pc-i440fx-1.7解释为QEMU模拟的是INTEL的i440fx硬件芯片组，1.7为QEMU的版本号。**找到默认机器之后，为其初始化物理内存**，QEMU首先**申请一块内存空间用于模拟虚拟机的物理内存空间**，申请完好内存之后，根据不同平台或者启动QEMU进程的参数，为虚拟机的**物理内存初始化**。具体函数调用过程见图1。

图 1. QEMU 硬件初始化函数调用流程图：

![config](images/2.jpg)

在QEMU中，整个物理内存以一个结构体struct MemoryRegion表示，具体定义见清单 5。

## 清单 5 QEMU 中 MemoryRegion 结构体

```
struct MemoryRegion {
    /* All fields are private - violators will be prosecuted */
    const MemoryRegionOps *ops;
    const MemoryRegionIOMMUOps *iommu_ops;
    void *opaque;
    struct Object *owner;
    MemoryRegion *parent;
    Int128 size;
    hwaddr addr;
    void (*destructor)(MemoryRegion *mr);
    ram_addr_t ram_addr;
    bool subpage;
    bool terminates;
    bool romd_mode;
    bool ram;
    bool readonly; /* For RAM regions */
    bool enabled;
    bool rom_device;
    bool warning_printed; /* For reservations */
    bool flush_coalesced_mmio;
    MemoryRegion *alias;
    hwaddr alias_offset;
    unsigned priority;
    bool may_overlap;
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) subregions_link;
    const char *name;
    uint8_t dirty_log_mask;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
    NotifierList iommu_notify;
};
```

**每一个MemoryRegion代表一块内存区域**。仔细观察 MemoryRegion 的成员函数，它包含一个Object的成员函数用于指向它的所有者，以及一个 MemoryRegion成员用于指向他的父节点（有点类似链表）。另外还有三个尾队列（QTAILQ） subregions， subregions\_link, subregions\_link。 也就是说，一个 MemoryRegion 可以包含多个内存区，根据不同的参数区分该内存域的功能。 在使用 MemoryRegion 之前要先为其分配内存空间并调用 memory\_region\_init 做必要的初始化。BIOS 也是通过一个 MemoryRegion 结构指示的。它的 MemoryRegion.name 被设置为"pc.bios"， size 设置为 BIOS 文件的大小（65536 的整数倍）。接着调用 rom\_add\_file\_fixed 将其 BIOS 文件加载到一个全局的 rom 队列中。

最后，回到 old\_pc\_system\_rom\_init 函数中，将 BIOS 映射到内存的最上方的地址空间。

## 清单 6 old\_pc\_system\_rom\_init函数中将 BIOS 映射到物理内存空间的代码：

```
hw/i386/pc_sysfw.c

    /* map all the bios at the top of memory */
    memory_region_add_subregion(rom_memory,
                                (uint32_t)(-bios_size),
                                bios);
```

(uint32\_t)(\-bios\_size) 是一个 32 位无符号数字，所以-bios\_size 对应的地址就是 FFFFFFFF 减掉 bios\_size 的大小。 bios size 大小为 ./pc-bios/bios.bin = 131072 （128KB）字节，十六进制表示为 0x20000，所以 bios 在内存中的位置为 bios position = fffe0000，bios 在内存中的位置就是 0xfffdffff~0xffffffff 现在 BIOS 已经加在到虚拟机的物理内存地址空间中了。

最后 QEMU 调用 CPU 重置函数重置 VCPU 的寄存器值 IP=0x0000fff0, CS=0xf000, CS.BASE= 0xffff0000,CS.LIMIT=0xffff. 指令从 0xfffffff0 开始执行，正好是 ROM 程序的开始位置。虚拟机就找到了 BIOS 的入口。

# 3 小结

通过阅读 QEMU 程序的源代码，详细介绍了 QEMU 中使用到的 BIOS 文件，QEMU 中物理内存的表示方法，以及 QEMU 是如何一步步将 BIOS 的二进制载入到通过 QEMU 创建的虚拟机中的内存的过程。

参考 http://wiki.qemu.org/Main_Page：关于 QEMU 项目的介绍。

参考 git://git.qemu.org/qemu.git 中 QEMU 的源代码。

Ubuntu 12.04之找不到Qemu命令 http://www.linuxidc.com/Linux/2012-11/73419.htm

Arch Linux上安装QEMU+EFI BIOS http://www.linuxidc.com/Linux/2013-02/79560.htm

QEMU的翻译框架及调试工具 http://www.linuxidc.com/Linux/2012-09/71211.htm

QEMU 的详细介绍：[请点这里](https://www.linuxidc.com/Linux/2013-08/88894.htm)

QEMU 的下载地址：[请点这里](https://www.linuxidc.com/down.aspx?id=960)

