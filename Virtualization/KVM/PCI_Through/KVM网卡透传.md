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

4, 