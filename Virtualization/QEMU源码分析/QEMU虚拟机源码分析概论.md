
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 QEMU官网描述](#1-qemu官网描述)
* [2 QEMU与KVM的关系](#2-qemu与kvm的关系)
* [3 QEMU的代码结构](#3-qemu的代码结构)

<!-- /code_chunk_output -->

# 1 QEMU官网描述

QEMU is a generic and open source machine emulator and virtualizer.

Full-system emulation: Run operating systems for any machine, on any supported architecture

User-mode emulation: Run programs for another Linux/BSD target, on any supported architecture

Virtualization: Run KVM and Xen virtual machines with near native performance

以上说明QEMU的以下特点:

1. QEMU既可以作为模拟器(emulator), 也可作为虚拟机(virtualizer)

2. 全系统模拟器: 在所有支持的架构上运行任何机器的操作系统

3. 用户模式模拟器: 在任何支持的架构上, 运行另一个Linux / BSD目标的程序

4. 虚拟机: 必须基于Xen Hypervisor或KVM内核模块才能支持虚拟化. 在这种条件下QEMU虚拟机可以通过直接在本机CPU上运行客户机代码获得接近本机的性能。

# 2 QEMU与KVM的关系

当QEMU在**模拟器模式**下，运行操作系统时，我们可以认为这是一种**软件实现的虚拟化技术**，它的效率比真机差很多，用户可以明显地感觉出来。

当QEMU在**虚拟机模式**下，QEMU必须在Linux上运行，并且需要借助**KVM或者Xen**，利用Intel或者Amd提供的硬件辅助虚拟化技术，才能使虚拟机达到接近真机的性能。

QEMU与KVM内核模块协同工作，在虚拟机进程中，各司其职，又相互配合，最终实现高效的虚拟机应用。

QEMU与KVM的关系如下：

![](./images/2019-06-03-10-02-47.png)

**KVM**在**物理机启动**时创建/**dev/kvm设备文件**，

当**创建虚拟机**时，KVM为**该虚拟机进程**创建一个**VM的文件描述符**，

当**创建vCPU**时，KVM为**每个vCPU**创建一个**文件描述符**。

同时，KVM向用户空间提供了一系列针对**特殊设备文件的ioctl系统调用**。QEMU主要是通过**ioctl系统调用**与KVM进行交互的。

那么QEMU和KVM具体都实现了哪些功能呢？我们用一张图说明：

![](./images/2019-06-03-10-18-43.png)

QEMU所实现的功能包括：虚拟机的配置和创建、虚拟机运行依赖的虚拟设备、虚拟机运行时用户操作环境和交互（vnc）以及一些针对虚拟机的特殊技术（如**动态迁移**），都是QEMU自己实现的。

同时QEMU还实现了利用KVM提供的接口实现虚拟机硬件加速。 

而KVM的主要功能在于初始化CPU硬件，打开虚拟化模式，然后将虚拟客户机运行在虚拟机模式下，并对虚拟客户机的运行提供支持，这些支持主要是以针对相关的特殊设备文件的ioctl系统调用。外设的模拟一般不会由KVM负责，只有对性能要求较高的虚拟设备，如虚拟中断控制器和虚拟时钟，是由KVM模拟的，这样可以大量减少处理器的**模式转换的开销**。

# 3 QEMU的代码结构











 参考

- QEMU官网: https://www.qemu.org/

