
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
* [2 什么是FirewallD](#2-什么是firewalld)
* [3 什么是iptables](#3-什么是iptables)
* [4 FirewallD服务的基本操作](#4-firewalld服务的基本操作)
* [5 iptables服务的基本操作](#5-iptables服务的基本操作)
* [6 理解网络区](#6-理解网络区)
	* [6.1 zone使用原则](#61-zone使用原则)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 概述

**防火墙**是一种位于**内部网络**与**外部网络**之间的**网络安全系统**。一项信息安全的防护系统，依照特定的规则，允许或是限制传输的数据通过。防火墙通常工作在**网络层**，也即**IPv4或IPv6的IP包**上。

是否允许包通过防火墙，取决于**防火墙配置的规则**。这些规则既可以是内建的，也可以是用户自定义的。每一个包要进出防火墙，均需要满足防火墙配置的规则。

**每一条规则**均有**一个目标动作**，具有**相同动作的规则**可以**分组在一起**。

RHEL7 中有几种防火墙共存：**firewalld**、**iptables**、**ebtables**等等，**默认**使用 **firewalld** 作为防火墙，**管理工具**是**firewall\-cmd**。使用 **firewalld** 来管理**netfilter**, 不过底层调用的命令仍然是 iptables等。

最常用的防火墙有：FirewallD或iptables。Linux的发行版种类极多，但是公认的仍然是这两种。

RedhatEnterprise Linux7 已经默认使用firewalld 作为防火墙，其使用方式已经变化。基于 iptables 的防火墙被默认不启动，但仍然可以继续使用。

因为这几种是冲突的，所以建议禁用其他几种服务。

# 2 什么是FirewallD

FirewallD即**Dynamic Firewall Manager of Linux systems**，Linux系统的**动态防火墙管理器**。是redhat7系统中对于netfilter内核模块的管理工具.

**iptables** service 管理防火墙规则的模式（**静态**）: 用户将新的防火墙规则添加进 /**etc/sysconfig/iptables** 配置文件当中，再执行命令 /etc/init.d/**iptables reload** 使变更的规则生效。在这整个过程的背后，iptables service 首先**对旧的防火墙规则进行了清空**，然后**重新完整地加载**所有新的防火墙规则，如果加载了防火墙的模块，需要在重新加载后进行手动加载防火墙的模块；

**firewalld** 管理防火墙规则的模式（**动态**）: 任何规则的变更都**不需要**对整个防火墙规则列表进行**重新加载**，只需要将**变更部分**保存并更新到运行中的 **iptables** 即可。还有命令行和图形界面配置工具，它仅仅是替代了 iptables service 部分，其底层还是使用 iptables 作为防火墙规则管理入口。

firewalld 使用 python 语言开发，在新版本中已经计划使用 c\+\+ 重写 daemon 部分(见下图)。

**FirewallD**是一个服务，用于**配置网络连接**，从而那些内外部网络的数据包可以允许穿过网络或阻止穿过网络。

**FirewallD**允许**两种类型的配置**：**永久类型**和**运行时类型**。

- 运行时类型的配置在**防火墙被重启后会丢失**相应的规则配置；
- 而永久类型的配置即使遇到系统重启，也会保留生效。

对应于上面两种类型的配置，FirewallD相应的有两个目录：

- 针对**运行时**类型配置的目录/**usr/lib/firewall**；
- 以及针对**永久类型**配置的目录/**etc/firewall**.

在RHEL/CentOS 7或Fedora 18的默认服务可以看到。

防火墙栈的整体图如下： 

![](./images/2019-04-30-22-37-15.png)

Firewalld 提供了支持**网络/防火墙区域(zone**)定义**网络链接**以及**接口安全等级**的**防火墙管理工具**。拥有运行时配置和永久配置选项。它也支持允许服务或者应用程序直接添加防火墙规则的接口。以前的 system\-config\-firewall 防火墙模型是**静态**的，每次修改都要求**防火墙完全重启**。这个过程包括**内核 netfilter 防火墙模块**的**卸载**和新配置所需模块的**装载**等。相反，firewalldaemon 动态管理防火墙，不需要重启整个防火墙便可应用更改。因而也就没有必要重载所有内核防火墙模块了。


查看防火墙几种服务的运行状态：

# 3 什么是iptables

iptables是另一种服务，它可以决定是否允许、删除或返回IP数据包。iptables服务管理IPv4数据包，而ip6tables则管理IPv6数据包。此服务管理了一堆规则表，其中每个表分别用于维护不同的目的，比如过滤表（filter table）为防火墙规则，NAT表供新连接查询使用，mangle表用于数据包的转换等。

更进一步，每个表还具有规则链，规则链可以是内建的或是用户自定义的，它表示适用于一个数据包的规则集合，从而决定数据包应该执行哪些目标动作，比如允许ALLOWED、阻塞BLOCKED或返回RETURNED。iptables服务在RHEL/CentOS 6/5、Fedora、ArchLinux、Ubuntu等Linux发行版中是系统默认的服务。

# 4 FirewallD服务的基本操作

对于CentOS/RHEL 7或Fedora 18以上版本的系统，要管理FirewallD服务，使用如下命令。

启动FirewallD服务

```
# systemctl firewalld start
```

停止FirewallD服务

```
# systemctl firewalld stop
```


检查FirewallD服务的状态

```
# systemctl status firewalld
```

检查FirewallD服务的状态

```
# firewall-cmd --state
```

可能会返回running，表示正在运行。

还可以禁用FirewallD服务，关闭那些规则。

禁用FirewallD服务

```
# systemctl disable firewalld
```

启用FirewallD服务

```
# systemctl enable firewalld
```

屏蔽FirewallD服务

```
# systemctl mask firewalld
```

还可以通过创建一个firewall.service到/dev/null的符号连接来屏蔽防火墙服务。

反屏蔽FirewallD服务

```
# systemctl unmask firewalld
```

这是反屏蔽FirewallD服务，它会移除屏蔽FirewallD服务时创建的符号链接，故能重新启用服务。

检查是否已安装防火墙

```
# yum install firewalld firewall-config
```

# 5 iptables服务的基本操作

在RHEL/CentOS 6/5/4系统和Fedora 12-18系统中，iptables是默认的防火墙，如果服务不存在，可以这样安装：

```
# yum install iptables-services
```

然后就可以对iptables服务进行启动、停止、重启等操作了。

启动iptables服务

```
# systemctl start iptables
```

或

```
# service iptables start
```

停止iptables服务

```
# systemctl stop iptables
```

或
```
# service iptables stop
```

禁用iptables服务

```
# systemctl disable iptables
```

或
```
# service iptables save
# service iptables stop
```

启用iptables服务
```
# systemctl enable iptables
```

或

```
# service iptables start
```

检查iptables服务的状态

```
# systemctl status iptables
```
或

```
# service iptables status
```

在Ubuntu及其它Linux发行版中，ufw是用于管理iptables防火墙服务的工具。ufw提供了一个简易的界面让用户可以很方便的处理iptables防火墙服务。

启用ufw iptables防火墙服务

```
$ sudo ufw enable
```

禁用ufw iptables防火墙服务

```
$ sudo ufw disable
```

检查ufw iptables防火墙服务的状态

```
$ sudo ufw status 
```

但是，如果想列出iptables包含的所有规则链列表，应使用如下命令：

```
$ iptables -L -n -v
```

# 6 理解网络区

网络区域定义了网络连接的可信等级。

![](./images/2019-05-03-14-15-45.png)

**数据包**要进入到**内核**必须要通过**这些zone中的一个**，而**不同的zone**里定义的**规则不一样**（即信任度不一样，过滤的强度也不一样）。可以根据网卡所连接的网络的安全性来判断，这张网卡的流量到底使用哪个 zone，比如上图来自 eth0 的流量全部使用 zone1 的过滤规则，eth1的流量使用 zone2。**一张网卡同时只能绑定到一个 zone**

在CentOS/RHEL 7系统中，基于用户对网络中**设备和通信**所给与的**信任程度**，防火墙可用于将**网络划分成不同的区域**，区域类型如下：

- 丢弃区域（Drop Zone）：

如果使用丢弃区域，任何进入的数据包将被丢弃。这个类似与我们之前使用iptables -j drop。使用丢弃规则意味着将不存在响应。

任何接收的网络数据包都被丢弃，没有任何回复。仅能有发送出去的网络连接。

- 阻塞区域（Block Zone）：

阻塞区域会拒绝进入的网络连接，返回icmp-host-prohibited，只有服务器已经建立的连接会被通过即只允许由该系统初始化的网络连接。

任何接收的网络连接都被 IPv4 的 icmp-host-prohibited 信息和 IPv6 的 icmp6-adm-prohibited 信息所拒绝。

- 公共区域（Public Zone）：

只接受那些被选中的连接，默认只允许 ssh 和 dhcpv6-client。这个 zone 是缺省 zone

在公共区域内使用，不能相信网络内的其他计算机不会对您的计算机造成危害，只能接收经过选取的连接。

- 外部区域（External Zone）：

这个区域相当于路由器的启用伪装（masquerading）选项。只有指定的连接会被接受，即 ssh，而其它的连接将被丢弃或者不被接受。

特别是为路由器启用了伪装功能的外部网。您不能信任来自网络的其他计算机，不能相信它们不会对您的计算机造成危害，只能接收经过选择的连接。

- dmz（非军事区） 隔离区域（DMZ Zone）：

如果想要只允许给部分服务能被外部访问，可以在 DMZ 区域中定义。它也拥有只通过被选中连接的特性，即 ssh。

用于您的非军事区内的电脑，此区域内可公开访问，可以有限地进入您的内部网络，仅仅接收经过选择的连接。

- 工作区域（Work Zone）：

在这个区域，我们只能定义内部网络。比如私有网络通信才被允许，只允许ssh，ipp-client 和 dhcpv6-client。

用于工作区。您可以基本相信网络内的其他电脑不会危害您的电脑。仅仅接收经过选择的连接。

- 家庭区域（Home Zone）：

这个区域专门用于家庭环境。它同样只允许被选中的连接，即 ssh，ipp-client，mdns，samba-client和 dhcpv6-client。

用于家庭网络。您可以基本信任网络内的其他计算机不会危害您的计算机。仅仅接收经过选择的连接。

- 内部区域（Internal Zone）：

这个区域和工作区域（Work Zone）类似，只有通过被选中的连接，和 home 区域一样。

用于内部网络。您可以基本上信任网络内的其他计算机不会威胁您的计算机。仅仅接受经过选择的连接。

- 信任区域（Trusted Zone）：

信任区域允许所有网络通信通过。记住：因为 trusted 是最被信任的，即使没有设置任何的服务，那么也是被允许的，因为trusted 是允许所有连接的

可接受所有的网络连接。

以上是系统定义的所有的 zone，但是这些 zone 并不是都在使用。只有活跃的 zone 才有实际操作意义。

对于区域的修改，可使用网络管理器NetworkManager搞定。

不同的区域之间的差异是其对待数据包的默认行为不同，firewalld的默认区域为public；

## 6.1 zone使用原则

Firewalld 的原则：

如果一个**客户端访问服务器**，服务器根据**以下原则**决定使用**哪个 zone** 的策略去匹配

1. 如果一个**客户端数据包**的**源 IP 地址**匹配 **zone** 的 **sources**，那么**该zone** 的规则就**适用这个客户端**；**一个源**只能属于**一个 zone**，不能同时属于多个 zone。

2. 如果一个**客户端数据包**进入服务器的**某一个接口（如eth0**）区配 **zone** 的interfaces，则么**该 zone 的规则**就适用这个客户端；一个接口只能属于一个 zone，不能同时属于多个zone。

3. 如果上述两个原则都不满足，那么**缺省的 zone** 将被应用

你可以使用任何一种 firewalld 配置工具来配置或者增加区域，以及修改配置。

工具有例如firewall\-config这样的图形界面工具， firewall\-cmd 这样的命令行工具，或者你也可以在配置文件目录中创建或者拷贝区域文件，/**usr/lib/firewalld/zones** 被用于**默认和备用配置**，/**etc/firewalld/zones**被用于**用户创建和自定义**配置文件。

文件：

- /usr/lib/firewalld/services/ ：firewalld服务默认在此目录下定义了70\+种服务供我们使用，格式：服务名.xml；
- /etc/firewalld/zones/ : 默认区域配置文件，配置文件中指定了编写完成的规则（规则中的服务名必须与上述文件名一致）；

分为多个文件的优点 :

第一，通过服务名字来管理规则更加人性化，

第二，通过服务来组织端口分组的模式更加高效，如果一个服务使用了若干个网络端口，则服务的配置文件就相当于提供了到这些端口的规则管理的批量操作快捷方式；



# 参考

- Linux基础——Firewalld防火墙: https://cloud.tencent.com/developer/article/1152579
- CentOS 7防火墙服务FirewallD指南: https://www.linuxidc.com/Linux/2016-10/136431.htm
- Centos7-----firewalld详解: https://blog.51cto.com/11638832/2092203