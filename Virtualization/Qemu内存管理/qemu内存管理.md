
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Qemu内存分布](#1-qemu内存分布)
* [2 内存初始化](#2-内存初始化)

<!-- /code_chunk_output -->

# 1 Qemu内存分布

![](./images/2019-06-15-22-08-00.png)

# 2 内存初始化

Qemu中的内存模型，简单来说就是Qemu申请用户态内存并进行管理，并将该部分申请的内存注册到对应的加速器（如KVM）中。这样的模型有如下好处：

策略与机制分离。加速的机制由KVM负责，而如何调用加速的机制由Qemu负责

可以由Qemu设置多种内存模型，如UMA、NUMA等等

方便Qemu对特殊内存的管理（如MMIO）

内存的分配、回收、换出等都可以采用Linux原有的机制，不需要为KVM单独开发。

兼容其他加速器模型（或者无加速器，单纯使用Qemu做模拟）

Qemu需要做的有两方面工作：向KVM注册用户态内存空间，申请用户态内存空间。

Qemu主要通过如下结构来维护内存：

参考

https://blog.51cto.com/zybcloud/2149626