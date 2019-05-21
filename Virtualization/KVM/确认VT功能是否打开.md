
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 VT\-x技术](#1-vt-x技术)

<!-- /code_chunk_output -->

Intel硬件虚拟化技术大致分为如下3个类别（这个顺序也基本上是相应技术出现的时间先后顺序）。

# 1 VT\-x技术

1）**VT\-x技术**：是指Intel处理器中进行的一些虚拟化技术支持，包括**CPU**中引入的最基础的**VMX技术**，使得KVM等硬件虚拟化基础的出现成为可能。同时也包括**内存虚拟化**的硬件支持**EPT、VPID**等技术。



2）**VT\-d**技术：是指Intel的**芯片组(南桥)的虚拟化**技术支持，通过Intel **IOMMU**可以实现对**设备直接分配**的支持。

VT-d技术可下载<Intel Virtualization Technology for Directed I/O Architecture Specification> 文档

3）**VT\-c**技术：是指Intel的**I/O设备**相关的虚拟化技术支持，主要包含**两个技术**：

- 一个是借助**虚拟机设备队列（VMDq**）最大限度提高I/O吞吐率，VMDq由**Intel网卡！！！** 中的**专用硬件**来完成；
- 另一个是借助**虚拟机直接互连（VMDc**）大幅提升虚拟化性能，VMDc主要就是**基于SR\-IOV标准**将**单个Intel网卡**产生**多个VF设备**，用来**直接分配**给客户机。