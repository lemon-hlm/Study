http://www.lenky.info/archives/2018/12/2667

# 1 KVM里CentOS 7虚拟机的网络设置

宿主机是CentOS 7，KVM虚拟机也是CentOS 7

在宿主机上有物理网卡eth0，配置有ip：192.168.1.2/24

有网桥virbr0，配置有ip：192.168.122.1/24

该virbr0绑定在virbr0\-nic接口上，而virbr0\-nic貌似是一个tun/tap设备，因此性能非常差。

所有KVM虚拟机挂在这个virbr0网桥上，然后通过virbr0-nic进行相互通信。
如果虚拟机要访问外部主机，则通过virbr0-nic做NAT出去。
如果外部主机要访问虚拟机，则比较麻烦，或许可能不行。
# brctl show
bridge name	bridge id	STP enabled	interfaces
virbr0	8000.52540098e452	yes	virbr0-nic
vnet0
vnet0是KVM虚拟机网卡在宿主机上对应的tap设备，如果还有其他KVM虚拟机，则都接到这个virbr0。