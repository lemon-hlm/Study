[TOC]

参考

```
https://www.ibm.com/support/knowledgecenter/linuxonibm/liaat/liaatbppassthrougtask.htm
http://www.linux-kvm.org/page/How_to_assign_devices_with_VT-d_in_KVM
```


为了提高虚拟机中的io吞吐量业界现在采用的方式就是直接将物理设备给虚拟机直接使用。

方式上有两种

- pci assign
- vfio

不过现在业界已经基本都采用vfio的方式了。主要是因为在vfio方式下对虚拟设备的权限和dma隔离上做的更好。

qemu直接使用物理设备本身命令行是很简单的，关键在于事先在主机上对系统、内核和物理设备的一些配置。

## 1. 最后要执行的QEMU命令

```
qemu-system-x86_64 -m 4096 -smp 4 --enable-kvm \
-drive file=~/guest/fedora.img \
-device vfio-pci,host=0000:00:01.0
```

其实和普通虚拟机启动就差了最后那个-device的选项。这个选项也比较容易理解，就是把主机上的设备0000:00:01.0传给了虚拟机使用。

## 2. 系统及硬件准备

### 2.1 BIOS中打开IOMMU

设备直通在x86平台上需要打开iommu功能。这是Intel虚拟技术VT-d(Virtualization Technology for Device IO)中的一个部分。有时候这部分的功能没有被打开。

打开的方式在BIOS设置中Security->Virtualization->VT-d这个位置。当然不同的BIOS位置可能会略有不同。记得在使用直通设备前要将这个选项打开。

### 2.2 内核配置勾选IOMMU

对应的在内核编译过程中需要勾选IOMMU的功能。在menuconfig中有这样的显示。

```
INTEL_IOMMU 
│ Location: │ 
│ -> Device Drivers │ 
│ (2) -> IOMMU Hardware Support (IOMMU_SUPPORT [=y])
```

如果你看到内核编译选项INTEL_IOMMU被设置为y，则表明已经配置成功。

### 2.3 内核启动参数enable IOMMU

BIOS中打开，内核编译选项勾选还不够。还需要在引导程序中添加上内核启动参数 intel\_iommu=on.

此处读者可能会将intel-iommu与iommu混淆，前者控制的是基于Intel VT-d的IOMMU，它可以使系统进行设备的DMA地址重映射（DMAR）等多种高级操作为虚拟机使用做准备，且此项默认关闭，而后者主要控制是GART（Graphics Address Remapping Table） IOMMU，目的是让有32位内存访问大小的设备可以进行DMAR操作，通常用于USB设备、声卡、集成显卡等，会在主机内存3GB以上的系统中默认开启。

### 2.4 确认IOMMU功能确实打开

所有的准备工作做完，重启主机后为了保证一切无误，可以采用如下方法检查是否已经enable了IOMMU.

```
dmesg | grep -e DMAR -e IOMMU
```

如果能搜索到

```
DMAR: IOMMU enabled
```

表示上述配置成功。

## 3. VFIO相关模块配置

### 3.1 vfio的相关配置选项

在x86平台，设备直通需要使用内核中vfio相关的模块。随意要确保内核配置选项中，相关的模块已编译。

相关的选项有

- VFIO
- VFIO-PCI
- VFIO_IOMMU_TYPE1

### 3.2 加载并确认vfio相关模块已经加载

如果上述选项设置是built-in的，那就不需要加载了。如果是模块的话，那需要加载并且这样还能确认是否选项勾选正确。

vfio\-pci驱动是专门为现在支持DMAR和中断地址重映射的PCI设备开发的驱动模块，它依赖于VFIO驱动框架，并且借助于vfio\-iommu\-type1模块实现IOMMU的重用

加载的命令：

```
modprobe vfio
modprobe vfio-pci
```

查看是否加载成功的命令是：

```
$ lsmod | grep vfio
vfio_pci               45056  0
vfio_virqfd            16384  1 vfio_pci
vfio_iommu_type1       20480  0
vfio                   28672  2 vfio_iommu_type1,vfio_pci
irqbypass              16384  2 kvm,vfio_pci
```

如果没有看到相关的模块加载，就需要重复上一个步骤重新配置并编译。

## 4. 设备配置

### 4.1 使能VF （可选）

#### 4.1.1 背景知识

VF字面上的理解就是虚拟设备。实际上是通过硬件和软件手段，使得同样的一个物理设备在系统上可以展现出多个逻辑设备。

然而这毕竟是一个高级功能 

* 首先是要硬件本身支持，就好像这高级本事要孙悟空级别的人物才有 
* 其次我们用的时候需要使能，就好像孙悟空平时也就是一个真身

有兴趣的同学可以搜索SRIOV关键字去找相关资料。如果希望再深入理解的话，恐怕要找SRIOV的规范来看了。我在这里尽量用通俗的语言解释。

现在使能某个设备的VF已经变得很简单了，找到带有SRIOV功能的设备，向其驱动中sysfs的某个文件echo你想要使能的VF个数即可。这里还要强调一下，被直通的设备可以是VF(virtual function)，也可以是PF(physical function)。

#### 4.1.2 找到PF

```
# lspci -vvv | grep "Single"
```

如果能找到，则表示系统中有PF，否则没有PF就不能使能VF。

记住拥有这个属性的PCI设备编号，假设编号为0000:00:01.0。

#### 4.1.3 使能VF

自从有了sysfs接口，使能VF变得简洁多了。方法就是向需要使能的PF设备的sysfs文件写入一个需要使能的VF个数。

假设PF的设备编号为0000:00:01.0， 想要使能10个VF

```
echo 10 > /sys/bus/pci/devices/0000:00:01.0/sriov_numvfs
```

#### 确认使能成功

一般情况下VF的设备名会带有”Virtual Function”字样，所以通过搜索lspci的输出可以确认是否使能成功。

```
lspci | grep "Virtual Function"
```

如果相对于使能前多出了相同指定个数的pci设备，则表示使能成功。

### 4.2 找到同一iommu group上所有设备

接下来的几个小节适用于所有的想要直通给虚拟机的pci设备，不管是PF还是VF，都需要如此操作。

- 首先是找到目标设备所在的iommu group

- 然后将同一个组的所有设备都卸载原有驱动并绑定到vfio-pci驱动上。

找同一个组的设备也是通过sysfs

```
ls /sys/bus/pci/devices/0000:00:01.0/iommu_group/devices
```

这样就会列出这个组下所有的设备

### 4.3 解绑定原有驱动

找到了同一个组下的设备，需要一一对设备做解绑定驱动的操作。

```
echo 0000:00:01.0 > /sys/bus/pci/devices/0000:00:01.0/driver/unbind
```

### 4.4 绑定vfio-pci驱动

绑定驱动和解绑定的操作类似，但是多了一个获取设备厂商号和型号的动作。

先获取设备厂商号和型号

```
lspci -ns 0000:00:01.0
00:02.0 0300: 8086:0412 (rev 06)
```

再绑定

```
echo 8086 0412 > /sys/bus/pci/drivers/vfio-pci/new_id
```

大功告成～现在你就可以执行最开始的那段代码运行带有直通设备的虚拟机了～