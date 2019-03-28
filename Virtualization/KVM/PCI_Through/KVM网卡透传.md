http://www.lenky.info/archives/2018/12/2667

# 1 KVM里CentOS 7虚拟机的网络设置

1. 宿主机是CentOS 7，KVM虚拟机也是CentOS 7

2. 在宿主机上有物理网卡eth0，配置有ip：192.168.1.2/24

有网桥virbr0，配置有ip：192.168.122.1/24

该virbr0绑定在virbr0\-nic接口上，而virbr0\-nic貌似是一个tun/tap设备，因此性能非常差。

**所有KVM虚拟机**挂在这个**virbr0网桥**上，然后通过virbr0\-nic进行相互通信。

如果虚拟机要访问外部主机，则通过virbr0\-nic做NAT出去。

如果外部主机要访问虚拟机，则比较麻烦，或许可能不行。

```
# brctl show
bridge name	bridge id	STP enabled	interfaces
virbr0	8000.52540098e452	yes	virbr0-nic
vnet0
```

vnet0是KVM虚拟机网卡在宿主机上对应的tap设备，如果还有其他KVM虚拟机，则都接到这个virbr0。

3，在上一步中，看到**网桥**是绑定在**virbr0\-nic虚拟设备**上的，其实可以直接绑定在宿主机的物理网卡eth0上。

a, 新增网桥br0以及配置

```
# brctl addbr br0
# vi /etc/sysconfig/network-scripts/ifcfg-br0
# cat /etc/sysconfig/network-scripts/ifcfg-br0
TYPE=bridge
BOOTPROTO=none
IPADDR=192.168.1.205
NETMASK=255.255.0.0
GATEWAY=192.168.1.254
NM_CONTROLLED=no
DEFROUTE=yes
PEERDNS=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no

NAME=br0
DEVICE=br0
ONBOOT=yes
```

b, 修改eth0网卡配置，在最后一行加上BRIDGE=br0，表示将eth0桥接到br0：

```
# cat /etc/sysconfig/network-scripts/ifcfg-enp4s0f0
TYPE=Ethernet
BOOTPROTO=static
IPADDR=192.168.1.208
NETMASK=255.255.0.0
GATEWAY=192.168.1.254
NM_CONTROLLED=no
DEFROUTE=yes
PEERDNS=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=enp4s0f0
DEVICE=enp4s0f0
ONBOOT=yes
BRIDGE=br0
```

c, 重启网络，务必多执行一次start，因为eth0启用依赖于br0，有可能第一次启动会失败。如果恰好eth0是用来做远程的，则可能导致网络断掉。

```
# service network restart; service network start;
```

把启动的虚拟机在宿主机上的对应网卡绑定到这个br0上：

```
# brctl show
bridge name	bridge id	STP enabled	interfaces
br0	8000.000d48064198	no	enp6s0f0
virbr0	8000.52540098e452	yes	virbr0-nic
vnet0
# brctl delif virbr0 vnet0
# brctl show
bridge name	bridge id	STP enabled	interfaces
br0	8000.000d48064198	no	enp6s0f0
virbr0	8000.52540098e452	yes	virbr0-nic
# brctl addif br0 vnet0
# brctl show
bridge name	bridge id	STP enabled	interfaces
br0	8000.000d48064198	no	enp6s0f0
vnet0
virbr0	8000.52540098e452	yes	virbr0-nic
```

这种设置的虚拟机网络性能相比上一种要好，而性能更好的设置方式就是直接把物理网卡pass through到虚拟机。

当然可以基于ovs网桥去做, 见其他文章

4, 做物理网卡pass through需要宿主机的硬件支持和一些准备工作

a, 确认宿主机的硬件支持，主要是cpu和主板，这可以查看官方的硬件支持列表，或者在BIOS中查看相关选项。以Intel硬件为例，主要就是：

VT-x：处理器技术，提供内存以及虚拟机的硬件隔离，所涉及的技术有页表管理以及地址空间的保护。

VT-d：处理有关芯片组的技术，它提供一些针对虚拟机的特殊应用，如支持某些特定的虚拟机应用跨过处理器I/O管理程序，直接调用I/O资源，从而提高效率，通过直接连接I/O带来近乎完美的I/O性能。

VT-c：针对网络提供的管理，它可以在一个物理网卡上，建立针对虚拟机的设备队列。

VT-c是后面将提到的SR-IOV关系比较大，本小节只需验证VT-x和VT-d，一般在BIOS中Advanced下CPU和System或相关条目中设置，都设置为Enabled：

VT：Intel Virtualization Technology

VT-d：Intel VT for Directed I/O

VT-c：I/O Virtualization

b, 修改内核启动参数，使IOMMU生效，CentOS7上修改稍微不同：

```
# cat /etc/default/grub
…
GRUB_CMDLINE_LINUX=”crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet intel_iommu=on”
…
```


在GRUB_CMDLINE_LINUX后加上intel_iommu=on，其他的不动。先备份，再重新生成grub.cfg：
```
# cp /boot/grub2/grub.cfg ~/grub.cfg.bak
# grub2-mkconfig -o /boot/grub2/grub.cfg
# diff /boot/grub2/grub.cfg ~/grub.cfg.bak
```
可以diff比较一下参数是否正确加上。

重启机器后执行如下两条命令进行确认：
# find /sys/kernel/iommu_groups/ -type l
# dmesg | grep -e DMAR -e IOMMU

如果有输出，那就说明ok了。如果没有，那再验证BIOS、内核编译项、内核启动参数等是否没有正确配置。比如内核是否已经编译了IOMMO：
# cat /boot/config-3.10.0-862.el7.x86_64 |grep IOMMU
CONFIG_GART_IOMMU=y
# CONFIG_CALGARY_IOMMU is not set
CONFIG_IOMMU_HELPER=y
CONFIG_VFIO_IOMMU_TYPE1=m
CONFIG_VFIO_NOIOMMU=y
CONFIG_IOMMU_API=y
CONFIG_IOMMU_SUPPORT=y
CONFIG_IOMMU_IOVA=y
CONFIG_AMD_IOMMU=y
CONFIG_AMD_IOMMU_V2=m
CONFIG_INTEL_IOMMU=y

c, 找一个没用的网卡设备，因为pass through是虚拟机独占，所以肯定不能用远程ip所对应的网卡设备，否则远程网络就断了。比如我的远程网卡为enp4s0f0，那么我这里拿enp8s0f0作为pass through网卡。

通过ethtool查看网卡的bus信息：
# ethtool -i enp8s0f0 | grep bus
bus-info: 0000:08:00.0

解除绑定（注意里面的0000:08:00.0是上一步获得的bus信息）：
# lspci -s 0000:08:00.0 -n
08:00.0 0200: 8086:10c9 (rev 01)
# modprobe pci_stub
# echo 0000:08:00.0 > /sys/bus/pci/devices/0000\:08\:00.0/driver/unbind
# echo “8086 10c9″ > /sys/bus/pci/drivers/pci-stub/new_id

驱动确认（注意里面的：Kernel driver in use: pci-stub）：
# lspci -s 0000:08:00.0 -k
08:00.0 Ethernet controller: Intel Corporation 82576 Gigabit Network Connection (rev 01)
Subsystem: Intel Corporation Device 0000
Kernel driver in use: pci-stub
Kernel modules: igb

启动虚拟机：
kvm -name centos7 -smp 4 -m 8192 \
-drive file=/home/vmhome/centos7.qcow2,if=virtio,media=disk,index=0,format=qcow2 \
-drive file=/home/lenky/CentOS-7-x86_64-DVD-1804.iso,media=cdrom,index=1 \
-nographic -vnc :2 \
-net none -device pci-assign,host=0000:08:00.0

注意最后两个参数：
‘-net none’：告诉qemu不用模拟网卡设备
‘-device pci-assign,host=0000:08:00.0’：直接指定一个pci设备，对应的地址为宿主机上pci地址0000:08:00.0

执行上面命令，我这里出现一个错误：
kvm: -device pci-assign,host=0000:08:00.0: No IOMMU found. Unable to assign device “(null)”
kvm: -device pci-assign,host=0000:08:00.0: Device initialization failed.
kvm: -device pci-assign,host=0000:08:00.0: Device ‘kvm-pci-assign’ could not be initialized

然后我前面的配置都ok啊，经过搜索，问题在于最新的内核里，已建议废除KVM_ASSIGN机制，而只支持vfio，我这里查看CentOS 7的内核编译选项也果真如此：
# cat /boot/config-3.10.0-862.el7.x86_64 | grep KVM_DEVICE
# CONFIG_KVM_DEVICE_ASSIGNMENT is not set

所以换用vfio驱动。VFIO可以用于实现高效的用户态驱动。在虚拟化场景可以用于device passthrough。通过用户态配置IOMMU接口，可以将DMA地址空间映射限制在进程虚拟空间中。这对高性能驱动和虚拟化场景device passthrough尤其重要。相对于传统方式，VFIO对UEFI支持更好。VFIO技术实现了用户空间直接访问设备。无须root特权，更安全，功能更多。

重新解除绑定和再绑定：
# modprobe vfio
# modprobe vfio-pci
# lspci -s 0000:08:00.0 -n
08:00.0 0200: 8086:10c9 (rev 01)
# echo “8086 10c9″ > /sys/bus/pci/drivers/vfio-pci/new_id
# echo 0000:08:00.0 > /sys/bus/pci/devices/0000\:08\:00.0/driver/unbind
# echo 0000:08:00.0 > /sys/bus/pci/drivers/vfio-pci/bind
# lspci -s 0000:08:00.0 -k
08:00.0 Ethernet controller: Intel Corporation 82576 Gigabit Network Connection (rev 01)
Subsystem: Intel Corporation Device 0000
Kernel driver in use: vfio-pci
Kernel modules: igb

启动虚拟机：
kvm -name centos7 -smp 4 -m 8192 \
-drive file=/home/vmhome/centos7.qcow2,if=virtio,media=disk,index=0,format=qcow2 \
-drive file=/home/lenky/CentOS-7-x86_64-DVD-1804.iso,media=cdrom,index=1 \
-nographic -vnc :2 \
-net none -device vfio-pci,host=0000:08:00.0

这次一切OK，顺利启动并进入到CentOS 7虚拟机。

https://blog.csdn.net/leoufung/article/details/52144687

https://www.linux-kvm.org/page/10G_NIC_performance:_VFIO_vs_virtio

http://www.linux-kvm.org/page/How_to_assign_devices_with_VT-d_in_KVM