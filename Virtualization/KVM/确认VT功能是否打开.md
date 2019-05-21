
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 BIOS设置](#1-bios设置)
* [2 VT\-x技术](#2-vt-x技术)
	* [2.1 处理器检查](#21-处理器检查)
	* [2.2 系统检查](#22-系统检查)

<!-- /code_chunk_output -->

# 1 BIOS设置

下面以一台Intel Haswell\-UP平台的服务器为例，来说明在BIOS中的设置。

BIOS中Enabled的**VT**和**VT\-d**选项，如图3-2所示。
￼
![](./images/2019-05-15-09-02-49.png)

对于不同平台或不同厂商的BIOS，VT和VT\-d等设置的位置可能是不一样的，需要根据实际的硬件情况和BIOS中的选项来灵活设置。

设置好了VT和VT\-d的相关选项，保存BIOS的设置并退出，系统重启后生效。

Intel硬件虚拟化技术大致分为如下3个类别（这个顺序也基本上是相应技术出现的时间先后顺序）。

# 2 VT\-x技术

指Intel处理器中进行的一些虚拟化技术支持，包括**CPU**中引入的最基础的**VMX技术**，使得KVM等硬件虚拟化基础的出现成为可能。同时也包括**内存虚拟化**的硬件支持**EPT、VPID**等技术。

也就是处理器虚拟化和内存虚拟化

## 2.1 处理器检查

在Linux系统中，可以通过检查/proc/cpuinfo文件中的CPU特性标志（flags）来查看CPU目前是否支持硬件虚拟化。

在x86和x86\-64平台中，**Intel**系列**CPU支持虚拟化**的标志为“**vmx**”，**AMD**系列CPU的标志为“**svm**”。

所以可以用以下命令行查看“vmx”或者“svm”标志：

```
[root@kvm-host ~]# grep -E "svm|vmx" /proc/cpuinfo
```

对于内存虚拟化**EPT**以及**vpid**的支持查询

```
[root@kvm-host ~]# grep -E “ept|vpid” /proc/cpuinfo ￼
```

如果查找到了表示你当前的CPU是**支持虚拟化功能**的，但是**不代表你现在的VT功能是开启**的。

## 2.2 系统检查

```
[root@kvm-host kvm]# modprobe kvm￼
[root@kvm-host kvm]# modprobe kvm_intel￼
[root@kvm-host kvm]# lsmod | grep kvm￼
kvm_intel             192512  0 ￼
kvm                   577536  1 kvm_intel
```

确认KVM相关的模块加载成功后，检查/**dev/kvm**这个文件，它是kvm内核模块提供给用户空间的qemu-kvm程序使用的一个控制接口，它提供了客户机（Guest）操作系统运行所需要的模拟和实际的硬件设备环境。

如果你当前该模块没有挂载的话，查看

可以尝试modprobe kvm\_intel如果提示你挂载失败，那么你当前的VT功能就是没有开启，需要你进入BIOS然后在CPU相关的配置项中进行开启，然后启动后再查看



2）**VT\-d**技术：是指Intel的**芯片组(南桥)的虚拟化**技术支持，通过Intel **IOMMU**可以实现对**设备直接分配**的支持。

VT-d技术可下载<Intel Virtualization Technology for Directed I/O Architecture Specification> 文档

3）**VT\-c**技术：是指Intel的**I/O设备**相关的虚拟化技术支持，主要包含**两个技术**：

- 一个是借助**虚拟机设备队列（VMDq**）最大限度提高I/O吞吐率，VMDq由**Intel网卡！！！** 中的**专用硬件**来完成；
- 另一个是借助**虚拟机直接互连（VMDc**）大幅提升虚拟化性能，VMDc主要就是**基于SR\-IOV标准**将**单个Intel网卡**产生**多个VF设备**，用来**直接分配**给客户机。