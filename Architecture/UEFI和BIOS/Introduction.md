UEFI大事年表

90年代初：传统BIOS的天下。计算机系统的创新严重受限。

1997年：英特尔开始为安腾电脑设计BIOS。

1999年：EFI 1.0 推出。

2000年：EFI 1.02发布。

2002年：EFI 1.10发布。

2005年：UEFI论坛成立。

2006年：UEFI 2.0发布。

2007年：UEFI 2.1

2008年：UEFI 2.2

## 基本概念

新型UEFI，全称“统一的可扩展固件接口”(Unified Extensible Firmware Interface)，是一种详细描述类型接口的标准。这种接口用于操作系统自动从预启动的操作环境，加载到一种操作系统上。

可扩展固件接口（Extensible Firmware Interface，EFI）是 Intel 为 PC 固件的体系结构、接口和服务提出的建议标准。其主要目的是为了提供一组在OS加载之前（启动前）在所有平台上一致的、正确指定的启动服务，被看做是有近20多年历史的 BIOS 的继任者。

UEFI是由EFI1.10为基础发展起来的，它的所有者已不再是Intel，而是一个称作Unified EFI Form的国际组织。

## 区别

与legacy BIOS（传统BIOS）相比，UEFI最大的几个区别在于：
1. 编码99%都是由C语言完成；

2. 一改之前的中断、硬件端口操作的方法，而采用了Driver/protocol的新方式；

3. 将不支持X86实模式，而直接采用Flat mode（也就是不能用DOS了，现在有些 EFI 或 UEFI 能用是因为做了兼容，但实际上这部分不属于UEFI的定义了）；

4. 输出也不再是单纯的二进制code，改为Removable Binary Drivers；

5. OS启动不再是调用Int19，而是直接利用protocol/device Path；

6. 对于第三方的开发，前者基本上做不到，除非参与BIOS的设计，但是还要受到ROM的大小限制，而后者就便利多了。

7. 弥补BIOS对新硬件的支持不足的问题。
