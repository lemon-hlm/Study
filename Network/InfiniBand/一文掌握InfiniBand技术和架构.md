https://cloud.tencent.com/developer/news/384580

Infiniband开放标准技术简化并加速了**服务器之间的连接**,同时支持**服务器**与**远程存储和网络设备的连接(！！！**)。OpenFabrics Enterprise Distribution (**OFED**)是一组开源软件驱动、核心内核代码、中间件和**支持InfiniBand Fabric的用户级接口程序**。

2005年由OpenFabrics Alliance(OFA)发布第一个版本。**Mellanox OFED**用于Linux，Windows (WinOF)，包括各种诊断和性能工具，用于**监视InfiniBand网络**的运行情况，包括监视传输带宽和监视Fabric内部的拥塞情况。

OpenFabrics Alliance (OFA)是一个基于开源的组织，它开发、测试、支持OpenFabrics企业发行版。该联盟的任务是开发并推广软件，通过将高效消息、低延迟和最大带宽技术架构直接应用到最小CPU开销的应用程序中，从而实现最大应用效率。

![config](./images/8.jpeg)

该联盟成立于2004年6月，最初是OpenIB联盟，致力于开发独立于供应商、基于Linux的InfiniBand软件栈。2005，联盟致力于支持Windows，此举将使软件栈真正跨平台。

2006年，该组织再次扩展其章程，包括对iWARP的支持，在2010年增加了对RoCE (RDMA over Converged)支持通过以太网交付高性能RDMA和内核旁路解决方案。2014年，随着OpenFabrics Interfaces工作组的建立，联盟再次扩大，实现对其他高性能网络的支持

# 1 IB技术的发展

1999年开始起草规格及标准规范，2000年正式发表，但发展速度不及Rapid I/O、PCI-X、PCI-E和FC，加上Ethernet从1Gbps进展至10Gbps。所以直到2005年之后，InfiniBand Architecture(IBA)才在集群式超级计算机上广泛应用。全球HPC高算系统TOP500大效能的超级计算机中有相当多套系统都使用上IBA。

除了InfiniBand Trade Association (IBTA)9个主要董事成员CRAY、Emulex、HP、IBM、intel、Mellanox、Microsoft、Oracle、Qlogic专门在应用和推广InfiniBand外，其他厂商正在加入或者重返到它的阵营中来，包括Cisco、Sun、NEC、LSI等。InfiniBand已经成为目前主流的高性能计算机互连技术之一。为了满足HPC、企业数据中心和云计算环境中的高I/O吞吐需求，新一代高速率56Gbps的FDR (Fourteen Data Rate) 和100Gpb EDR InfiniBand技术已经广泛应用。

# 2 IB技术的优势

Infiniband大量用于FC/IP SAN、NAS和服务器之间的连接,作为iSCSI RDMA的存储协议iSER已被IETF标准化。目前EMC全系产品已经切换到Infiniband组网，IBM/TMS的FlashSystem系列，IBM的存储系统XIV Gen3，DDN的SFA系列都采用Infiniband网络。

相比FC的优势主要体现在性能是FC的3.5倍，Infiniband交换机的延迟是FC交换机的1/10，支持SAN和NAS。

存储系统已不能满足于传统的FC SAN所提供的服务器与裸存储的网络连接架构。HP SFS和IBM GPFS 是在Infiniband fabric连接起来的服务器和iSER Infiniband存储构建的并行文件系统，完全突破系统的性能瓶颈。

Infiniband采用PCI串行高速带宽链接，从SDR、DDR、QDR、FDR到EDR HCA连接，可以做到1微妙、甚至纳米级别极低的时延，基于链路层的流控机制实现先进的拥塞控制。

InfiniBand采用虚通道(VL即Virtual Lanes)方式来实现QoS，虚通道是一些共享一条物理链接的相互分立的逻辑通信链路，每条物理链接可支持多达15条的标准虚通道和一条管理通道(VL15)

![config](./images/9.jpeg)

RDMA技术实现内核旁路，可以提供远程节点间RDMA读写访问，完全卸载CPU工作负载，基于硬件传出协议实现可靠传输和更高性能。

![config](./images/10.jpeg)

相比TCP/IP网络协议，IB使用基于信任的、流控制的机制来确保连接的完整性，数据包极少丢失，接受方在数据传输完毕之后，返回信号来标示缓存空间的可用性，所以IB协议消除了由于原数据包丢失而带来的重发延迟，从而提升了效率和整体性能。

TCP/IP具有转发损失的数据包的能力，但是由于要不断地确认与重发，基于这些协议的通信也会因此变慢，极大地影响了性能。

# 3 IB基本概念

IB是以通道为基础的双向、串行式传输，在连接拓朴中是采用交换、切换式结构(Switched Fabric)，在线路不够长时可用IBA中继器(Repeater)进行延伸。每一个IBA网络称为子网(Subnet)，每个子网内最高可有65,536个节点(Node)，IBA Switch、IBARepeater仅适用于Subnet范畴，若要通跨多个IBASubnet就需要用到IBA路由器(Router)或IBA网关器(Gateway)。

每个节点(Node) 必须透过配接器(Adapter)与IBA Subnet连接，节点CPU、内存要透过HCA(Host Channel Adapter)连接到子网；节点硬盘、I/O则要透过TCA(TargetChannel Adapter)连接到子网，这样的一个拓扑结构就构成了一个完整的IBA。

IB的传输方式和介质相当灵活，在设备机内可用印刷电路板的铜质线箔传递(Backplane背板)，在机外可用铜质缆线或支持更远光纤介质。若用铜箔、铜缆最远可至17m，而光纤则可至10km，同时IBA也支持热插拔，及具有自动侦测、自我调适的Active Cable活化智能性连接机制。

# 4 IB协议简介

InfiniBand也是一种**分层协议(类似TCP/IP协议**)，每层负责不同的功能，下层为上层服务，不同层次相互独立。 IB采用IPv6的报头格式。其数据包报头包括本地路由标识符LRH，全局路由标示符GRH，基本传输标识符BTH等。

