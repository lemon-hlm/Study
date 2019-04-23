
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
* [2 为什么需要片内总线？](#2-为什么需要片内总线)
* [3 星型连接](#3-星型连接)
* [4 环形总线(Ring Bus)](#4-环形总线ring-bus)
* [5 Mesh网路](#5-mesh网路)
* [6 结论](#6-结论)
* [7 参考](#7-参考)

<!-- /code_chunk_output -->

# 1 概述

在大多数普通用户眼里，CPU也许就是一块顶着铁盖子的电路板而已。但是如果我们揭开顶盖，深入正中那片小小的集成电路，我们会发现这里有着人类科技史上，最伟大的奇迹。几十亿个晶体管层层叠叠，密密麻麻挤在一起，占据着这个仅有一点几个平方厘米的狭小世界。**晶体管**们在“上帝之手”的安排下，组成了**各个功能模块**。而这些**功能模块之间**，则是宽阔无比的**超高速公路**。这条超高速公路如此重要，以至于它的距离、速度和塞车与否，会大大影响这个小小世界的效率。

这些**模块**就是**CPU内部的功能模块**，例如**内存控制器！！！**、**L3/L2 Cache**、**PCU**、**PCIe root complex！！！**等等，而这条高速公路，就是我们今天的主角，片内总线了。

# 2 为什么需要片内总线？

片内总线**连接Die内部**的**各个模块(！！！各个模块！！！**)，是它们之间信息交流的必须途径。它的设计高效与否，会大大影响CPU的性能。如果我们把各个模块看做一个个节点，那么它们之间相互连接关系一般可以有以下几种：

![](./images/2019-04-23-12-19-07.png)

而我们CPU片内总线连接就经历了从**星型（Star**）连接 \-\-> **环形总线**（Ring Bus）\-\-> **Mesh网络**的过程。

# 3 星型连接

早期CPU内部**模块数目较少**，结构单一，星型连结不失是一个简单高效的办法。

![](./images/2019-04-23-12-20-43.png)

我们尊贵的**Core**无可争议的被放在**中心的位置**，各个模块都和它连接，而彼此并不直接交互，必须要**Core中转**。这种设计简单高效，被使用了相当长的时间。

问题随着**多core**的出现而显现出来，这个多出来的core放在哪里合适呢？一种星型连接的变种被利用起来。它类似一种**双星结构**，中间的节点似乎进行了有丝分裂，一分为二，各自掌管着自己的势力范围。同时为了高效，每个Core又伸出了些触手，和别的Core的小弟发生了些不清不楚的关系。问题被暂时解决了，这种混乱的关系被固定下来，世界又恢复了些许和平，直到**Core数目的进一步增加**。

# 4 环形总线(Ring Bus)

Intel的服务器产品线是第一个受不了这种临时安排的。**至强CPU**必须提供**更多的CPU**，而**低效的变种星形连结**限制了内核数目的增加，加上各个模块之间通讯的需求越来越多，一种新的总线便孕育而出。

在**Nehalem EP/EX**这个划时代的产品中，很多新技术被引入进来，Intel也由此确定了领先的地位。而**Ring Bus**就是其中重要的一个。

![](./images/2019-04-23-12-22-43.png)

我们可以看到，Ring Bus实际上是**两个环**，一个**顺时针环**和一个**逆时针环**。**各个模块**一视同仁的通过**Ring Stop**挂接在**Ring Bus**上。如此设计带来很多好处：

1. **双环**设计可以保证**任何两个ring stop之间距离不超过Ring Stop总数的一半**，延迟较低。
2. 各个模块之间交互方便，**不需要Core中转**。这样一些高级加速技术，如DCA（Direct Cache Access), Crystal Beach等等应运而生。
3. **高速的ring bus**保证了性能的极大提高。Core to Core latency只有**60ns**左右，而带宽则高达数百G(记得Nehalem是192GB/s).
4. 方便灵活。**增加一个Core**，只要在Ring上面加个新的ring stop就好，不用像以前一样考虑复杂的互联问题。

Intel Xeon E5 v4 Low Core Count(LCC)

![](./images/2019-04-23-13-08-28.png)

真是个绝妙的设计！然而，**摩尔定律**是无情的，计划能使用好久的设计往往很快就过时了，这点在计算机界几乎变成了普遍规律。

Ring Bus的缺点也很快随着内核的快速增加而暴露出来。由于**每加一个Core**，**ring bus**就长大一些，**延迟就变大一点**，很快ring bus性能就随着core的增多而严重下降，**多于12个core**后会严重拖累系统的**整体延迟**。

和星型连接一样，一种变种产生了：

![](./images/2019-04-23-13-09-27.png)

Intel Xeon E5-2600 V4 **High Core Count** Die(HCC)

注: **只有第一个NUMA节点**才连接QPI Link、PCIE、UBox和PCU, 但是每个都连接了Memory Controller

在**至强HCC(High Core Count, 核很多版**)版本中，又加入了一个ring bus。

**两个ring bus**各接**12个Core**，将**延迟**控制在可控的范围内。**俩个Ring Bus**直接用两个**双向Pipe Line连接**，保证通讯顺畅。于此同时由于Ring 0中的模块访问Ring 1中的模块延迟明显高于本Ring，亲缘度不同，所以**两个Ring分属于不同的NUMA！！！**（Non\-Uniform Memory Access Architecture）node。这点在 **BIOS设计！！！** 中要特别注意。

聪明的同学可能要问了，如果Core比12个多，比24个少些呢？是不是凑合塞到第一个ring里拉倒呢？其实还有个1.5 ring的奇葩设计：

Intel Xeon E5 V4 MCC

![](./images/2019-04-23-13-12-07.png)

核大战的硝烟远远尚未平息，摩尔定律带来的晶体管更多的都用来增加内核而不是提高速度（[为什么CPU的频率止步于4G?我们触到频率天花板了吗？](https://zhuanlan.zhihu.com/p/30409360)）24个Core的至强也远远不是终点，那么更多的Core怎么办呢？三个Ring设计吗？多于3个Ring后，它们之间怎么互联呢？这些困难促使Intel寻找新的方向。

# 5 Mesh网路

Intel在**Skylake**和**Knight Landing**中引入了新的**片内总线**：Mesh。

它是一种2D的Mesh网络：

Intel Skylake SP Mesh Architecture Conceptual Diagram:

![](./images/2019-04-23-13-00-25.png)

![](./images/2019-04-23-13-07-23.png)

Mesh网络近几年越来越火热，它的灵活性吸引越来越多的产品加入对它的支持，包括我们的Wifi等等系统。Mesh网络引入片内总线是一个巨大的进步，它有很多优点：

1. 首先当然是灵活性。新的模块或者节点在Mesh中增加十分方便，它带来的延迟不是像ring bus一样线性增加，而是非线性的。从而可以容纳更多的内核。
2. 设计弹性很好，不需要1.5 ring和2ring的委曲求全。
3. 双向mesh网络减小了两个node之间的延迟。过去两个node之间通讯，最坏要绕过半个ring。而mesh整体node之间距离大大缩减。
4. 外部延迟大大缩短：

⓵ RAM延迟大大缩短：

Broadwell Ring V Skylake Mesh DRAM Example:

![](./images/2019-04-23-13-23-35.png)

左边的是ring bus，从**一个ring**里面访问**另一个ring**里面的**内存控制器！！！**。最坏情况下是那条绿线，拐了一个大圈才到达内存控制器，需要310个cycle。而在Mesh网络中则路径缩短很多。

⓶ IO延迟缩短:

Broadwell Ring V Skylake Mesh PCIe Example:

![](./images/2019-04-23-13-25-30.png)

注: 在过去的几代中，英特尔一直在使用QuickPath Interconnect（QPI）作为高速点对点互连。 QPI已被Ultra Path Interconnect（UPI）取代，后者是可扩展系统的更高效的一致性互连，允许多个处理器共享一个共享地址空间。根据确切的型号，**每个处理器**可以有**两个或三个UPI链接**连接到**其他处理器**。

# 6 结论

Mesh网络带来了这么多好处，那么缺点有没有呢？它网格化设计带来复杂性的增加，从而对Die的大小带来了负面影响，这个我们会在下一篇文章中介绍，同时介绍相关性能详细数据，尽情期待。

最后请大家思考一下，为什么不干脆用全互联Full Connected网络来连接Die中的各个节点呢？

![](./images/2019-04-23-13-26-16.png)

# 7 参考

- 本文章来自: https://zhuanlan.zhihu.com/p/32216294
- Xeon\_e5的wikichip: https://en.wikichip.org/wiki/intel/xeon_e5
- Mesh Interconnect Architecture-Intel: https://en.wikichip.org/wiki/intel/mesh_interconnect_architecture
- Things are getting Meshy: Next-Generation Intel Skylake-SP CPUs Mesh Architecture: https://www.servethehome.com/things-are-getting-meshy-next-generation-intel-skylake-sp-cpus-mesh-architecture/
- Intel Core i9 7900X review: the best around, but the worst time to buy a high-end CPU: https://www.pcgamesn.com/intel/intel-core-i9-7900x-review-benchmarks