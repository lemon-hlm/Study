
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->



<!-- /code_chunk_output -->

虚拟机运行概览

首先直观的了解一下利用QEMU运行客户虚拟机的流程。

在命令行中运行QEMU的启动命令格式举例：

```
qemu-system-x86_64 --enable-kvm -cpu host \ 
-smp cores=4,threads=2,sockets=4 \
-m 16384 -k en-us -hda /pps/guohongwei/vm_test/ubuntu.img -monitor stdio
```

QEMU的核心初始化流程

客户系统运行之前，QEMU作为全系统模拟软件，需要为客户系统模拟出CPU、主存以及I/O设备，使客户系统就像运行在真实硬件之上，而不用对客户系统代码做修改。

如概览部分所示，由用户为客户系统指定需要的虚拟CPU资源（包括CPU核心数，SOCKET数目，每核心的超线程数，是否开启NUMA等等），虚拟内存资源，具体参数设置参见${QEMU}/qemu-options.hx。创建QEMU主线程，执行QEMU系统的初始化，在初始化的过程中针对每一个虚拟CPU，单独创建一个posix线程。每当一个虚拟CPU线程被调度到物理CPU上执行时，该VCPU对应的一套完整的寄存器集合被加载到物理CPU上，通过VM-LAUNCH或VM-RESUME指令切换到非根模式执行。直到该线程时间片到，或者其它中断引发虚拟机退出，VCPU退出到根模式，进行异常处理。
如下图所示，当用户运行QEMU的System Mode的可执行文件时，QEMU从${QEMU}/vl.c的main函数执行主线程。以下着重分析，客户系统启动之前，QEMU所做的初始化工作：




参考

https://blog.csdn.net/lux_veritas/article/details/9383643