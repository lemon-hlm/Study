#### KVM基础

```
KVM网站：http://www.linux-kvm.org/page/Main_Page
KVM博客：http://blog.csdn.net/RichardYSteven/article/category/841588
```


KVM（Kernel-based Virtual Machine）是Linux下基于X86硬件（包含虚拟化扩展<Intel VT 或 AMD-V>）的全虚拟化解决方案。KVM包含一个可加载的内核模块（kvm.ko），这个模块提供了核心的虚拟化基础架构和一个特定的处理器模块（kvm-intel.ko或kvm-amd.ko）

使用KVM，可以运行多个虚拟机，这些虚拟机是未修改过的Linux或Windows镜像。每个虚拟机都有独自的虚拟硬件：网卡、磁盘、图形适配器等。

KVM是一个开源软件。KVM的核心组件被包含在Linux的主线版本中（2.6.20）。KVM的用户空间组件包含在QEMU的主线版本（1.3）。

活跃在KVM相关虚拟化发展的人们的博客被组织在http://planet.virt-tools.org/这个网站。



