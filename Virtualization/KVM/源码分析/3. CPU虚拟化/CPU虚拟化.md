
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1. VT-x 技术](#1-vt-x-技术)
* [2. VMCS寄存器](#2-vmcs寄存器)
* [3. VM-Entry/VM-Exit](#3-vm-entryvm-exit)

<!-- /code_chunk_output -->

## 1. VT-x 技术

Intel处理器支持的虚拟化技术即是VT-x，之所以CPU支持硬件虚拟化是因为软件虚拟化的效率太低。

处理器虚拟化的本质是分时共享，主要体现在状态恢复和资源隔离，实际上每个VM对于VMM看就是一个task么，之前Intel处理器在虚拟化上没有提供默认的硬件支持，传统 x86 处理器有4个特权级，Linux使用了0,3级别，0即内核，3即用户态，（[更多参考CPU的运行环、特权级与保护](http://blog.csdn.net/drshenlei/article/details/4265101)）而在虚拟化架构上，虚拟机监控器的运行级别需要内核态特权级，而CPU特权级被传统OS占用，所以Intel设计了VT-x，提出了VMX模式，即VMX root operation 和 VMX non-root operation，虚拟机监控器运行在VMX root operation，虚拟机运行在VMX non-root operation。每个模式下都有相对应的0~3特权级。

为什么引入这两种特殊模式，在传统x86的系统中，CPU有不同的特权级，是为了划分不同的权限指令，某些指令只能由系统软件操作，称为特权指令，这些指令只能在最高特权级上才能正确执行，反之则会触发异常，处理器会陷入到最高特权级，由系统软件处理。还有一种需要操作特权资源（如[访问中断寄存器](http://www.oenhan.com/rwsem-realtime-task-hung)）的指令，称为敏感指令。OS运行在特权级上，屏蔽掉用户态直接执行的特权指令，达到控制所有的硬件资源目的；而在虚拟化环境中，VMM控制所有所有硬件资源，VM中的OS只能占用一部分资源，OS执行的很多特权指令是不能真正对硬件生效的，所以原特权级下有了root模式，OS指令不需要修改就可以正常执行在特权级上，但这个特权级的所有敏感指令都会传递到root模式处理，这样达到了VMM的目的。

在KVM源代码分析1:基本工作原理章节中也说了kvm分3个模式，对应到VT-x 中即是客户模式对应vmx非root模式，内核模式对应VMX root模式下的0特权级，用户模式对应vmx root模式下的3特权级。

![config](images/1.png)

在非根模式下敏感指令引发的陷入称为VM-Exit，VM-Exit发生后，CPU从非根模式切换到根模式；对应的，VM-Entry则是从根模式到非根模式，通常意味着调用VM进入运行态。VMLAUCH/VMRESUME命令则是用来发起VM-Entry。

## 2. VMCS寄存器

VMCS保存虚拟机的相关CPU状态，每个VCPU都有一个VMCS（内存的），每个物理CPU都有VMCS对应的寄存器（物理的），当CPU发生VM-Entry时，CPU则从VCPU指定的内存中读取VMCS加载到物理CPU上执行，当发生VM-Exit时，CPU则将当前的CPU状态保存到VCPU指定的内存中，即VMCS，以备下次VMRESUME。

VMLAUCH指VM的第一次VM-Entry，VMRESUME则是VMLAUCH之后后续的VM-Entry。VMCS下有一些控制域：

![config](images/2.png)

关于具体控制域的细节，还是翻Intel手册吧。

## 3. VM-Entry/VM-Exit

VM-Entry是从根模式切换到非根模式，即VMM切换到guest上，这个状态由VMM发起，发起之前先保存VMM中的关键寄存器内容到VMCS中，然后进入到VM-Entry，VM-Entry附带参数主要有3个：1.guest是否处于64bit模式，2.MSR VM-Entry控制，3.注入事件。1应该只在VMLAUCH有意义，3更多是在VMRESUME，而VMM发起VM-Entry更多是因为3，2主要用来每次更新MSR。

VM-Exit是CPU从非根模式切换到根模式，从guest切换到VMM的操作，VM-Exit触发的原因就很多了，执行敏感指令，[发生中断](http://www.oenhan.com/rwsem-realtime-task-hung)，模拟特权资源等。

运行在非根模式下的敏感指令一般分为3个方面：

1.行为没有变化的，也就是说该指令能够正确执行。

2.行为有变化的，直接产生VM-Exit。

3.行为有变化的，但是是否产生VM-Exit受到VM-Execution控制域控制。

主要说一下"受到VM-Execution控制域控制"的敏感指令，这个就是针对性的硬件优化了，一般是1.产生VM-Exit；2.不产生VM-Exit，同时调用优化函数完成功能。典型的有“RDTSC指令”。除了大部分是优化性能的，还有一小部分是直接VM-Exit执行指令结果是异常的，或者说在[虚拟化](http://www.oenhan.com/kvm-src-1)场景下是不适用的，典型的就是TSC offset了。

VM-Exit发生时退出的相关信息，如退出原因、触发中断等，这些内容保存在VM-Exit信息域中。

