
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 概述

Intel从奔腾4开始的CPU中增加了一种机制，称为MCA——Machine Check Architecture，它用来检测硬件（这里的Machine表示的就是硬件）错误，比如系统总线错误、ECC错误等等。

这套系统通过一定数量的MSR（Model Specific Register）来实现，这些MSR分为两个部分，一部分用来进行设置，另一部分用来描述发生的硬件错误。

当CPU检测到不可纠正的MCE（Machine Check Error）时，就会触发#MC（Machine Check Exception），通常软件会注册相关的函数来处理#MC，在这个函数中会通过读取MSR来收集MCE的错误信息，然后重启系统。当然由于发生的MCE可能是非常致命的，CPU直接重启了，没有办法完成MCE处理函数；甚至有可能在MCE处理函数中又触发了不可纠正的MCE，也会导致系统直接重启。

当然CPU还会检测到可纠正的MCE，当可纠正的MCE数量超过一定的阈值时，会触发CMCI（Corrected Machine Check Error Interrupt），此时软件可以捕捉到该中断并进行相应的处理。CMCI是在MCA之后才加入的，算是对MCA的一个增强，在此之前软件只能通过轮询可纠正MCE相关的MSR才能实现相关的操作。


Intel在Pentium 4、Xenon和P6系列处理器中实现了机器检查（Machinecheck）架构，提供能够检测和报告硬件（机器）的错误机制，如系统总线错误、ECC错误、奇偶校验错误、缓存错误、TLB错误等。它包括一直MSR（Model-Specific Registers）寄存器，用来设置机器检查和额外的bank MSR记录错误。

当机器检查到不可纠正的machine\-check错误时，就触发一个machine\-check异常。machine\-check架构不允许在出现MCE后处理器重启，但MCE处理程序可以从MSR寄存器收集相关信息。


IA32\_MCG\_STATUS MSR描述了发生Machine Check exception后处理器的当前状态.

![](./images/2019-04-28-11-53-49.png)

- bit 0: 设置后说明在生成机器检查异常时，可以在堆栈上按下的指令指针指向的指令处可靠地重新启动程序执行。 清零时，程序无法在推送的指令指针处可靠地重新启动。
- bit 1: 设置后说明生成机器检查异常时指令指针指向的指令与错误直接关联。 清除此标志时，指向的指令可能与错误无关。
- bit 2: 设置后说明生成机器检查异常。 软件可以设置或清除此标志。 设置MCIP时发生第二次机器检查事件将导致处理器进入关闭状态。 
- bit 3: 设置后说明生成本地machine\-check exception. 这表示当前的机器检查事件仅传递给此逻辑处理器。

![](./images/2019-04-28-12-14-19.png)

bd80000000100134的二进制如下:

```
1011 1101 1000 0000 0000 0000 0000 0000 
0000 0000 0001 0000 0000 0001 0011 0100
```

Bit 63: VAL. 表示本寄存器中包含有效的错误码
Bit 61: UC. 表示是无法纠正的MCE
Bit 60: EN. 表示处于允许报告错误的状态
Bit 59: MISCV. 表示MCi_MISC寄存器中含有对该错误的补充信息
Bit 58: ADDRV. 表示MCi_ADDR寄存器含有发生错误的内存地址
Bit 56: 发出未校正的可恢复（UCR）错误信号
Bit 55: UCR错误所需的恢复操作
Bit[16: 31]: 特定CPU型号相关的扩展错误码. 本例中是0x0010.
Bit[0: 15]: MCE错误码, 该错误码是所有CPU型号通用的，分为两类：simple error codes（简单错误码） 和 compound error codes（复合错误码），本例中0x0134表示Memory errors in the cache hierarchy(缓存层次结构中的内存错误)：

- Simple Error Codes:

![](./images/2019-04-28-12-38-31.png)

- Compound Error Codes:

![](./images/2019-04-28-12-38-58.png)


Bit 12: 0代表正常过滤

2位TT子字段（表15-11）表示事务的类型（数据，指令或通用）。 子字段适用于TLB，高速缓存和互连错误条件。 请注意，互连错误条件主要与P6系列和奔腾处理器相关，后者使用独立于系统总线的外部APIC总线。 当处理器无法确定事务类型时，将报告泛型类型。这里是Data.

![](./images/2019-04-28-12-47-33.png)

2位LL子字段（参见表15-12）指示发生错误的存储器层次结构中的级别（级别0，级别1，级别2或通用）。 LL子字段也适用于TLB，高速缓存和互连错误条件。 Pentium 4，Intel Xeon，Intel Atom和P6系列处理器支持缓存层次结构中的两个级别和TLB中的一个级别。 同样，当处理器无法确定层次结构级别时，将报告泛型类型。

![](./images/2019-04-28-12-49-48.png)




```
[root@SH-IDC1-10-5-8-61 yanbaoyue]# mcelog
Hardware event. This is not a software error.
MCE 0
CPU 19 BANK 14 TSC f3086e49d7de
RIP !INEXACT! 10:ffffffff816ab6a5
MISC 900010080000086 ADDR 5f14903680
TIME 1556364329 Sat Apr 27 19:25:29 2019
MCG status:RIPV MCIP
MCi status:
Error overflow
Uncorrected error
Error enabled
MCi_MISC register valid
MCi_ADDR register valid
SRAO
MCA: MEMORY CONTROLLER MS_CHANNEL1_ERR
Transaction: Memory scrubbing error
MemCtrl: Uncorrected patrol scrub error
STATUS fd001dc0001000c1 MCGSTATUS 5
MCGCAP f000814 APICID 60 SOCKETID 1
PPIN fd003dc0001000c1
CPUID Vendor Intel Family 6 Model 85
Hardware event. This is not a software error.
MCE 1
CPU 19 BANK 14 TSC f308a56e97a8
RIP !INEXACT! 10:ffffffff816ab6a5
MISC 900001001000086 ADDR 5f149ab080
TIME 1556364329 Sat Apr 27 19:25:29 2019
MCG status:RIPV MCIP
MCi status:
Error overflow
Uncorrected error
Error enabled
MCi_MISC register valid
MCi_ADDR register valid
SRAO
MCA: MEMORY CONTROLLER MS_CHANNEL1_ERR
Transaction: Memory scrubbing error
MemCtrl: Uncorrected patrol scrub error
STATUS fd003dc0001000c1 MCGSTATUS 5
MCGCAP f000814 APICID 60 SOCKETID 1
PPIN fd003a40001000c1
CPUID Vendor Intel Family 6 Model 85
Hardware event. This is not a software error.
MCE 2
CPU 43 BANK 14 TSC f308d5268140
RIP !INEXACT! 10:ffffffff816ab6a5
MISC 900100000000086 ADDR 5f14a5b480
TIME 1556364329 Sat Apr 27 19:25:29 2019
MCG status:RIPV MCIP
MCi status:
Error overflow
Uncorrected error
Error enabled
MCi_MISC register valid
MCi_ADDR register valid
SRAO
MCA: MEMORY CONTROLLER MS_CHANNEL1_ERR
Transaction: Memory scrubbing error
MemCtrl: Uncorrected patrol scrub error
STATUS fd003a40001000c1 MCGSTATUS 5
MCGCAP f000814 APICID 61 SOCKETID 1
CPUID Vendor Intel Family 6 Model 85

-----------------------
[166492.496342] mce: [Hardware Error]: CPU 19: Machine Check Exception: f Bank 1: bd80000000100134
[166492.496424] mce: [Hardware Error]: RIP 10:<ffffffffc1003eaa> {vvp_page_own+0xa/0xc0 [lustre]}
[166492.496541] mce: [Hardware Error]: TSC 1e4ee3d9dafdc ADDR 3b14a3c880 MISC 86
[166492.496609] mce: [Hardware Error]: PROCESSOR 0:50654 TIME 1556280681 SOCKET 1 APIC 60 microcode 2000057
[166492.496692] mce: [Hardware Error]: Machine check events logged
[166492.496767] mce: [Hardware Error]: Run the above through 'mcelog --ascii'
[166492.498942] Kernel panic - not syncing: Machine check from unknown source
```

# 参考

- x86架构——MCA: https://blog.csdn.net/jiangwei0512/article/details/62456226
