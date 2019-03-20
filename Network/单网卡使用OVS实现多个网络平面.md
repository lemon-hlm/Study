在 OVS 中, 有几个非常重要的概念：

- Bridge: Bridge 代表一个**以太网交换机（Switch**），一个主机中可以创建一个或者多个 Bridge 设备。
- Port: 端口与物理交换机的端口概念类似，**每个 Port** 都隶属于**一个 Bridge**。
- Interface: 连接到 Port 的网络接口设备。在通常情况下，Port 和 Interface 是一对一的关系, 只有在配置 Port 为 bond 模式后，Port 和 Interface 是一对多的关系。

# 1 OVS中的bridge

一个桥就是一个交换机。在OVS中，

```

```

```
[root@controller124 ~]# ovs-vsctl add-br br1
[root@controller124 ~]# ovs-vsctl add-br br2
[root@controller124 ~]# ovs-vsctl add-br br3
[root@controller124 ~]# ovs-vsctl add-br br4
[root@controller124 ~]# ovs-vsctl list-br
br-ex
br1
br2
br3
br4
[root@controller124 ~]# ovs-vsctl add-port br-ex patch-to-br1
[root@controller124 ~]# ovs-vsctl set interface patch-to-br1 type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br1 options:peer=patch-to-br1-brex
[root@controller124 ~]# ovs-vsctl add-port br1 patch-to-br1-brex
[root@controller124 ~]# ovs-vsctl set interface patch-to-br1-brex type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br1-brex options:peer=patch-to-br1
[root@controller124 ~]#
[root@controller124 ~]#
[root@controller124 ~]# ovs-vsctl add-port br-ex patch-to-br2
[root@controller124 ~]# ovs-vsctl set interface  patch-to-br2 type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br2 options:peer=patch-to-br2-brex
[root@controller124 ~]# ovs-vsctl add-port br2 patch-to-br2-brex
[root@controller124 ~]# ovs-vsctl set interface patch-to-br2-brex type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br2-brex options:peer=patch-to-br2
[root@controller124 ~]#
[root@controller124 ~]#
[root@controller124 ~]# ovs-vsctl add-port br-ex patch-to-br3
[root@controller124 ~]# ovs-vsctl set interface patch-to-br3 type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br3 options:peer=patch-to-br3-brex
[root@controller124 ~]# ovs-vsctl add-port br3 patch-to-br3-brex
[root@controller124 ~]# ovs-vsctl set interface patch-to-br3-brex type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br3-brex options:peer=patch-to-br3
[root@controller124 ~]#
[root@controller124 ~]#
[root@controller124 ~]# ovs-vsctl add-port br-ex patch-to-br4
[root@controller124 ~]# ovs-vsctl set interface patch-to-br4 type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br4 options:peer=patch-to-br4-brex
[root@controller124 ~]# ovs-vsctl add-port br4 patch-to-br4-brex
[root@controller124 ~]# ovs-vsctl set interface patch-to-br4-brex type=patch
[root@controller124 ~]# ovs-vsctl set interface patch-to-br4-brex options:peer=patch-to-br4

[root@controller124 ~]# ip address add  10.121.2.134/24 dev br1
[root@controller124 ~]# ip address add  10.121.2.135/24 dev br2
[root@controller124 ~]# ip address add  10.121.2.136/24 dev br3
[root@controller124 ~]# ip address add  10.121.2.137/24 dev br4
```