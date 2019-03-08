# qemu,kvm,qemu-kvm,xen,libvirt的区别

> 
参考
区别：https://my.oschina.net/qefarmer/blog/386843  
虚拟化之QEMU和KVM：http://blog.chinaunix.net/uid-23769728-id-3256677.html

## 虚拟化类型

- 全虚拟化（Full Virtualization)

全虚拟化也成为原始虚拟化技术，该模型使用虚拟机协调guest操作系统和原始硬件，VMM在guest操作系统和裸硬件之间用于工作协调，一些受保护指令必须由Hypervisor（虚拟机管理程序）来捕获处理。因为操作系统是通过Hypervisor来分享底层硬件.一些受保护的指令必须由Hypervisor(虚拟机管理程序)来捕获和处理. 因为操作系统是通过Hypervisor来分享底层硬件.

![全虚拟化模型](http://7u2ho6.com1.z0.glb.clouddn.com/tech-full-virtualization.gif)
全虚拟化模型：使用Hypervisor分享底层硬件

全虚拟化的运行速度要快于硬件模拟, 但是性能方面不如裸机, 因为Hypervisor需要占用一些资源. 全虚拟化最大的优点是操作系统没有经过任何修改. 它的唯一限制是操作系统必须能够支持底层硬件(比如, PowerPC).

一些老的硬件如x86, 全虚拟化遇到了问题. 比如, 一些敏感的指令需要由VMM来处理(VMM不能设置陷阱). 因此, Hypervisors必须动态扫描和捕获特权代码来处理问题.

- 半虚拟化（Para Virtualization）

半虚拟化是另一种类似于全虚拟化的技术，它使用Hypervisor分享存取底层的硬件，但是它的guest操作系统集成了虚拟化方面的代码。该方法无需重新编译或引起陷阱，因为操作系统自身能够与虚拟进程进行很好的协作。

![半虚拟化模型](http://7u2ho6.com1.z0.glb.clouddn.com/tech-para-virtualization.gif)
半虚拟化模型：通过客户操作系统分享进程

半虚拟化需要guest操作系统做一些修改，使guest操作系统意识到自己是处于虚拟化环境的，但是半虚拟化提供了与原操作系统相近的性能。半虚拟化需要客户操作系统做一些修改(配合Hypervisor), 这是一个不足之处. 但是半虚拟化提供了与原始系统相近的性能. 与全虚拟化一样, 半虚拟化可以同时能支持多个不同的操作系统.

## 虚拟化技术

### KVM

KVM：(Kernel-based Virtual Machine)基于内核的虚拟机

KVM是集成到Linux内核的Hypervisor，是X86架构且硬件支持虚拟化技术（Intel VT或AMD-V）的Linux的全虚拟化解决方案。它是Linux的一个很小的模块，利用Linux做大量的事，如任务调度、内存管理与硬件设备交互等。

![KVM虚拟化平台架构](http://7u2ho6.com1.z0.glb.clouddn.com/tech-kvm-architecture.jpg)
KVM虚拟化平台架构

从存在形式看,KVM是两个内核模块kvm.ko和kvm_intel.ko(对AMD处理器来说，就是kvm_amd.ko)，这两个模块用来实现CPU的虚拟化。 如果要让用户在KVM上完成一个虚拟机相关的操作，显然需要用户空间的东西，同时还包括IO虚拟化，所以KVM的解决方案借鉴了QEMU的东西并做了一定的修改，形成了自己的KVM虚拟机工具集和IO虚拟化的支持，也就是所谓的qemu-kvm.（KVM is a fork of QEMU, namely qemu-kvm)

## XEN

Xen是第一类运行在裸机上的虚拟化管理程序(Hypervisor)。它支持全虚拟化和半虚拟化,Xen支持hypervisor和虚拟机互相通讯，而且提供在所有Linux版本上的免费产品，包括Red Hat Enterprise Linux和SUSE Linux Enterprise Server。Xen最重要的优势在于半虚拟化，此外未经修改的操作系统也可以直接在xen上运行(如Windows)，能让虚拟机有效运行而不需要仿真，因此虚拟机能感知到hypervisor，而不需要模拟虚拟硬件，从而能实现高性能。 QEMU is used by Xen.

![Xen虚拟化平台架构](http://7u2ho6.com1.z0.glb.clouddn.com/tech-xen-architecture.jpg)
Xen虚拟化平台架构

## QEMU

QEMU是一套由Fabrice Bellard所编写的模拟处理器的自由软件。它与Bochs，PearPC近似，但其具有某些后两者所不具备的特性，如高速度及跨平台的特性，qemu可以虚拟出不同架构的虚拟机，如在x86平台上可以虚拟出power机器。kqemu为qemu的加速器，经由kqemu这个开源的加速器，QEMU能模拟至接近真实电脑的速度。

QEMU本身可以不依赖于KVM，但是如果有 KVM的存在并且硬件(处理器)支持比如Intel VT功能，那么QEMU在对处理器虚拟化这一块可以利用KVM提供的功能来提升性能。

QEMU目前最新的版本是1.1.0 ( http://www.qemu.org), download到本地后，比如放到/home/dennis/workspace/Linux下是qemu-1.1.0-1.tar.bz2, 解压之后得到一个qemu-1.1.0的目录，为了实验的目的，我不想让它替换掉我系统中已经安装的qemu工具集合（bin tools set)，所以我在/home/dennis /workspace/Linux下面重新建立了一个目录，比如qemu-bin-dbg. 先进入到/home/dennis/workspace/Linux/qemu-1.10目录下，然后依次执行：
1)./configure --prefix=/home/dennis/workspace/Linux/qemu-bin-dbg --enable-debug
这一步需要说明的是，我实验用的环境是ubuntu 12.04, 内核版本是我自己更新的3.4.3, intel i3处理器，在我实际操作过程中出现了以下的错误：

```
Error: zlib check failed
Make sure to have the zlib libs and headers installed.
```

google了一下，这个错误很容易解决：

```
apt-get install zlib1g-dev
apt-get install libglib2.0-dev
2) make
3) make install
``` 

然后我们到/home/dennis/workspace/Linux/qemu-bin-dbg目录下，会发现QEMU的工具集都放在了其中的bin目录下，其中会有qemu-i386和qemu-system-i386，前者只是对处理器的模拟(指令)，后者则模拟一个基于i386的PC. 如果没有KVM的支持，比如rmmod kvm_intel，然后再运行qemu-system-i386，那么就会出现这样的提示信息：

Could not access KVM kernel module: No such file or directory 
failed to initialize KVM: No such file or directory
Back to tcg accelerator.

因为前面我大体浏览过KVM的某些代码，KVM内核模块会生成一个“/dev/kvm”设备文件供用户空间程序使用，上面的提示信息就是因为QEMU在初始化阶段因为无法找到该“/dev/kvm"文件，因此认为当前系统没有提供对KVM的支持，因而QEMU回退到所谓的tcg accelerator模式，这从另一个角度说明，独立的QEMU虚拟化方案并不一定需要KVM提供支持。

我的理解是KVM模块的存在可以视为QEMU对i386处理器模拟的一种accelerator，如同GPU对3D的hardware acceleration一样(在没有GPU存在的情况下，软件也可以实现某些3D的功能，不过性能显然要慢多了，比如mesa).所以即便在kvm_intel.ko不存在的情况下，QEMU也可以完成对一个pc的虚拟化。

KVM是Kernel-based Virtual Machine,从存在形式看，是两个内核模块kvm.ko和kvm_intel.ko(对AMD处理器来说，就是kvm_amd.ko)，这两个模块用来实现CPU的虚拟化，如果要让用户在KVM上完成一个虚拟机相关的操作，显然需要用户空间的东西，同时还包括IO虚拟化，所以KVM的解决方案借鉴了QEMU的东西并做了一定的修改，形成了自己的KVM虚拟机工具集和IO虚拟化的支持，也就是所谓的qumu-kvm.

## KVM和QEMU的关系

准确来说，KVM是Linux kernel的一个模块。可以用命令modprobe去加载KVM模块。加载了模块后，才能进一步通过其他工具创建虚拟机。但仅有KVM模块是远远不够的，因为用户无法直接控制内核模块去做事情,你还必须有一个运行在用户空间的工具才行。这个用户空间的工具，kvm开发者选择了已经成型的开源虚拟化软件QEMU。说起来QEMU也是一个虚拟化软件。它的特点是可虚拟不同的CPU。比如说在x86的CPU上可虚拟一个Power的CPU，并可利用它编译出可运行在Power上的程序。KVM使用了QEMU的一部分，并稍加改造，就成了可控制KVM的用户空间工具了。所以你会看到，官方提供的KVM下载有两大部分(qemu和kvm)三个文件(KVM模块、QEMU工具以及二者的合集)。也就是说，你可以只升级KVM模块，也可以只升级QEMU工具。这就是KVM和QEMU的关系。

![KVM和QEMU关系](http://7u2ho6.com1.z0.glb.clouddn.com/tech-kvm-and-qemu.png)
KVM和QEMU关系

QEMU是个独立的虚拟化解决方案，从这个角度它并不依赖KVM。 而KVM是另一套虚拟化解决方案，不过因为这个方案实际上只实现了内核中对处理器(Intel VT, AMD SVM)虚拟化特性的支持，换言之，它缺乏设备虚拟化以及相应的用户空间管理虚拟机的工具，所以它借用了QEMU的代码并加以精简，连同KVM一起构成了另一个独立的虚拟化解决方案，不妨称之为：KVM+QEMU. 

KVM用户空间虚拟机管理工具有kvm， kvm-img， kvm-nbd ，kvm-ok 和kvm_stat，实际上kvm就是一个指向qemu-system-x86_64的符号链接，kvm-img则是指向qemu-img的符号链接。从适用的范围来讲，QEMU虚拟化方案除了支持x86架构外，还支持其他很多架构，比如qemu-system-m68k，qemu-system-mips64， qemu-system-ppc64， qemu-system-arm等等。但是目前提到KVM，一般指x86上基于Intel VT和AMD SVM的解决方案，虽然目前将KVM移植到ARM, PPC的工作正在进行中。

当然由于redhat已经开始支持KVM，它认为KVM+QEMU'的方案中用户空间虚拟机管理工具不太好使，或者通用性不强，所以redhat想了一个libvirt出来，一个用来管理虚拟机的API库，不只针对KVM,也可以管理Xen等方案下的虚拟机。

kvm-qemu可执行程序像普通Qemu一样：分配RAM,加载代码，不同于重新编译或者调用callingKQemu，它创建了一个线程（这个很重要）；这个线程调用KVM内核模块去切换到用户模式，并且去执行VM代码。当遇到一个特权指令，它重新切换回KVM内核模块，该内核模块在需要的时候，像Qemu线程发信号去处理大部分的硬件仿真。

这个体系结构一个比较巧妙的一个地方就是客户代码被模拟在一个posix线程，这允许你使用通常Linux工具管理。如果你需要一个有2或者4核的虚拟机，kvm-qemu创建2或者4个线程，每个线程调用KVM内核模块并开始执行。并发性（若果你有足够多的真实核）或者调度（如果你不管）是被通用的Linux调度器，这个使得KVM代码量十分的小。当一起工作的时候，KVM管理CPU和MEM的访问，QEMU仿真硬件资源（硬盘，声卡，USB，等等）当QEMU单独运行时，QEMU同时模拟CPU和硬件。

综上：QEMU虚拟化(模拟器)解决方案并不依赖KVM，而KVM虚拟化方案则一定要依赖QEMU(至少目前是），即便redhat开发了libvirt，但后者可以简单认为是一个虚拟机管理工具，libvirt依然需要用户空间的QEMU来和KVM交互。
