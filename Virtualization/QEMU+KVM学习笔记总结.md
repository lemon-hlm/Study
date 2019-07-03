
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [ 1 QEMU和KVM的关系](#1-qemu和kvm的关系)
- [ 2 QEMU基本介绍](#2-qemu基本介绍)
- [ 3 KVM基本介绍](#3-kvm基本介绍)
- [ 4 KVM架构](#4-kvm架构)
- [ 5 KVM工作原理](#5-kvm工作原理)

<!-- /code_chunk_output -->

# 1 QEMU和KVM的关系

现在所说的虚拟化，一般都是指在CPU硬件支持基础之上的虚拟化技术。KVM也同hyper\-V、Xen一样依赖此项技术。没有CPU硬件虚拟化的支持，KVM是无法工作的。

准确来说，KVM是Linux的一个模块。可以用modprobe去加载KVM模块。加载了模块后，才能进一步通过其他工具创建虚拟机。但仅有KVM模块是远远不够的，因为用户无法直接控制内核模块去作事情：还必须有一个**用户空间的工具**才行。这个用户空间的工具，开发者选择了已经成型的开源虚拟化软件 QEMU。说起来QEMU也是一个虚拟化软件。它的特点是可虚拟不同的CPU。比如说在x86的CPU上可虚拟一个Power的CPU，并可利用它编译出 可运行在Power上的程序。KVM使用了QEMU的一部分，并稍加改造，就成了可控制KVM的用户空间工具了。所以你会看到，官方提供的**KVM下载**有两大部分三个文件，分别是**KVM模块**、**QEMU工具**以及二者的合集。也就是说，你可以只升级KVM模块，也可以只升级QEMU工具。

# 2 QEMU基本介绍

Qemu是一个完整的可以单独运行的软件，它可以用来模拟机器，非常灵活和可移植。它主要通过一个特殊的'重编译器'将为特定处理器编写二进制代码转换为另一种。（也就是，在PPCmac上面运行MIPS代码，或者在X86 PC上运行ARM代码）

# 3 KVM基本介绍

KVM是一个基于Linux内核的虚拟机，它属于完全虚拟化范畴，从Linux-2.6.20开始被包含在Linux内核中。KVM基于x86硬件虚拟化技术，它的运行要求Intel VT-x或AMD SVM的支持。

一般认为，虚拟机监控的实现模型有两类：监控模型（Hypervisor）和宿主机模型（Host-based）。由于监控模型需要进行处理器调度，还需要实现各种驱动程序，以支撑运行其上的虚拟机，因此实现难度上一般要大于宿主机模型。KVM的实现采用宿主机模型（Host-based），由于KVM是集成在Linux内核中的，因此可以自然地使用Linux内核提供的内存管理、多处理器支持等功能，易于实现，而且还可以随着Linux内核的发展而发展。另外，目前KVM的所有I/O虚拟化工作是借助Qemu完成的，也显著地降低了实现的工作量。以上可以说是KVM的优势所在。

# 4 KVM架构

kvm基本结构有2个部分构成：

- kvm 驱动：现在已经是linux kernel的一个模块了。其主要负责虚拟机的创建，虚拟内存的分配，VCPU寄存器的读写以及VCPU的运行。

- Qemu：用于模拟虚拟机的用户空间组件，提供I/O设备模型，访问外设的途径。

![](./images/2019-07-03-21-38-22.png)

kvm基本结构如上图。kvm已经是内核模块，被看作是一个标准的linux 字符集设备（/dev/kvm）。Qemu通过libkvm应用程序接口，用fd通过ioctl向设备驱动来发送创建，运行虚拟机命令。设备驱动kvm就会来解析命令（kvm\_dev\_ioctl函数在kvm\_main.c文件中）,如下图：

![](./images/2019-07-03-21-38-48.png)

KVM模块让Linux主机成为一个虚拟机监视器（VMM，Virtual Machine Monitor），并且在原有的Linux两种执行模式基础上，新增加了客户模式，客户模式拥有自己的内核模式和用户模式。在虚拟机运行时，三种模式的工作各为：

- 客户模式：执行非I/O的客户代码，虚拟机运行在这个模式下。
- 用户模式：代表用户执行I/O指令，qemu运行在这个模式下。
- 内核模式：实现客户模式的切换，处理因为I/O或者其他指令引起的从客户模式退出（VM_EXIT）。kvm 模块工作在这个模式下。

在kvm的模型中，每一个Gust OS都是作为一个标准的linux进程，都可以使用linux进程管理命令管理。

KVM三种类型的文件描述符

首先是kvm设备本身。kvm内核模块本身是作为一个设备驱动程序安装的，驱动的设备名称是”/dev/kvm“。要使用kvm，需要先用open打开”/dev/kvm”设备，得到一个kvm设备文件描述符fd，然后利用此fd调用ioctl就可以向设备驱动发送命令了。kvm驱动解析此种请求的函数是kvm_dev_ioctl(kvm_main.c)，如KVM_CREATE_VM。

     其次是具体的VM。通过KVM_CREATE_VM创建了一个VM后，用户程序需要发送一些命令给VM，如KVM_CREATE_VCPU。这些命令当然也是要通过ioctl来发送，所以VM也需要对应一个文件描述符才行。用户程序中用ioctl发送KVM_CREATE_VM得到的返回值就是新创建VM对应的fd，之后利用此fd发送命令给此VM。kvm驱动解析此种请求的函数是kvm_vm_ioctl。此外，与OS线程类似，每个VM在kvm驱动中会对应一个VM控制块结构struct kvm，每个对VM的内核操作都基本要访问这个结构，那么kvm驱动是如何找到请求这次命令的VM的控制块的呢？回答这个问题首先要知道，linux内核用一个struct file结构来表示每个打开的文件，其中有一个void *private_data字段，kvm驱动将VM控制块的地址保存到对应struct file的private_data中。用户程序发送ioctl时，指定具体的fd，内核根据fd可以找到相应的struct file，传递给kvm_vm_ioctl，再通过private_data就可以找到了。

     最后是具体的VCPU。原理基本跟VM情况差不多，kvm驱动解析此种请求的函数是kvm_vcpu_ioctl。VCPU控制块结构为struct kvm_vcpu。

# 5 KVM工作原理
    KVM的基本工作原理：用户模式的Qemu利用接口libkvm通过ioctl系统调用进入内核模式。KVM Driver为虚拟机创建虚拟内存和虚拟CPU后执行VMLAUCH指令进入客户模式。装载Guest OS执行。如果Guest OS发生外部中断或者影子页表（shadow page）缺页之类的事件，暂停Guest OS的执行，退出客户模式进行一些必要的处理。然后重新进入客户模式，执行客户代码。如果发生I/O事件或者信号队列中有信号到达，就会进入用户模式处理。KVM采用全虚拟化技术。客户机不用修改就可以运行。