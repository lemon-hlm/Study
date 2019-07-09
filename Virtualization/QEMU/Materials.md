1. QOM模型

启动QEMU 增加参数：

```
-qmp unix:/tmp/qmp.socket,server,nowait
```

在QEMU源码中, 动态dump qom模型

```
# cd scripts/qmp/
# ./qom-tree -s /tmp/qmp.socket
```

2. VMX硬件支持情况

这是CPU支持的情况, 里面每一项都涉及到体系结构信息, 每个都是一个技术点

```
# cd scripts/kvm/
# ./vmxcap
```