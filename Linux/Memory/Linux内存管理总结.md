[TOC]

# 1 学习路线

很多Linux内存管理从**malloc**()这个C函数开始，从而知道**虚拟内存**。**虚拟内存是什么，怎么虚拟**？早期系统没有虚拟内存概念，**为什么**现代OS都有？要搞清楚虚拟内存，可能需要了解**MMU、页表、物理内存、物理页面、建立映射关系、按需分配、缺页中断和写时复制**等机制。

MMU，除了MMU工作原理，还会接触到Linux内核**如何建立页表映射**，其中也包括**用户空间页表的建立**和**内核空间页表**的建立，以及内核是如何**查询页表和修改页表**的。

当了解**物理内存**和**物理页面**时，会接触到**struct pg\_data\_t、struct zone和 struct page**等数据结构，这3个数据结构描述了系统中**物理内存的组织架构**。struct page数据结构除了描述一个4KB大小（或者其他大小）的物理页面外，还包含很多复杂而有趣的成员。

当了解**怎么分配物理页面**时，会接触到**伙伴系统机制**和**页面分配器**（**page allocator**),页面分配器是内存管理中最复杂的代码之一。

有了**物理内存**，那**怎么和虚拟内存建立映射关系**呢？在 Linux内核中，描述**进程的虚拟内存**用**struct vm\_area\_struct**数据结构。**虚拟内存**和**物理内存**采用**建立页表**的方法来**完成建立映射关系**。为什么**和进程地址空间建立映射的页面**有的叫**匿名页面**，而有的叫**page cache页面**呢？

当了解**malloc()怎么分配出物理内存**时，会接触到**缺页中断**，缺页中断也是内存管理中最复杂的代码之一。

这时，**虚拟内存和物理内存己经建立了映射关系**，这是**以页为基础**的，可是有时内核需要**小于一个页面**大小的内存，那么**slab机制**就诞生了。

上面己经建立起虚拟内存和物理内存的基本框图，但是如果用户**持续分配和使用内存**导致**物理内存不足**了怎么办？此时**页面回收机制**和**反向映射机制**就应运而生了。

虚拟内存和物理内存的映射关系经常是**建立后又被解除**了，时间长了，系统**物理页面布局变得凌乱**不堪，碎片化严重，这时内核如果需要**分配大块连续内存**就会变得很困难，那么**内存规整机制**（Memory Compaction) 就诞生了。

# 2 3种系统架构与2种存储器共享方式

从**系统架构**来看，目前的商用服务器大体可以分为三类

- 对称多处理器结构(SMP：Symmetric Multi\-Processor)

- 非一致存储访问结构(NUMA：Non\-Uniform Memory Access)

- 海量并行处理结构(MPP：Massive Parallel Processing)。

**共享存储型**多处理机有两种模型

- 均匀存储器存取（Uniform\-Memory\-Access，简称UMA）模型

- 非均匀存储器存取（Nonuniform\-Memory\-Access，简称NUMA）模型

## 2.1 SMP

服务器中**多个CPU对称工作**，**无主次或从属关系**。

**各CPU**共享**相同的物理内存**，**每个CPU**访问**内存中的任何地址**所需**时间是相同的**，因此SMP也被称为**一致存储器访问结构(UMA：Uniform Memory Access)**

![UMA多处理机模型如图所示](./images/5.gif)

**每个CPU**必须通过**相同的内存总线**访问**相同的内存资源**.

## 2.2 NUMA

![NUMA多处理机模型如图所示](./images/6.png)

![config](./images/7.gif)

NUMA服务器的基本特征是具有**多个CPU模块**，**每个CPU模块由多个CPU(如4个)组成**，并且具有独立的**本地内存、I/O槽口等**(这些**资源都是以CPU模式划分本地的，一个CPU模块可能有多个CPU！！！**)。

其**共享存储器**物理上是分布在**所有处理机的本地存储器上**。**所有本地存储器**的集合组成了**全局地址空间**，可被所有的处理机访问。**处理机**访问**本地存储器是比较快**的，但访问属于**另一台处理机的远程存储器**则比较**慢**，因为通过互连网络会产生附加时延。

## 2.3 MPP

其基本特征是由多个SMP服务器(**每个SMP服务器称节点**)通过**节点互联网络**连接而成，**每个节点只访问自己的本地资源**(内存、存储等)，是一种完全**无共享(Share Nothing)结构**，协同工作，**完成相同的任务**. **在MPP系统中，每个SMP节点也可以运行自己的操作系统、数据库等**。**每个节点内的CPU不能访问另一个节点的内存**。

目前业界对**节点互联网络暂无标准**

# 3 内存空间分层

内存空间分三层

![config](./images/3.jpg)

![config](./images/4.png)

| 层次 | 描述 |
|:---:|:----|
| **用户空间层** |可以理解为Linux 内核内存管理**为用户空间暴露的系统调用接口**. 例如**brk**(), **mmap**()等**系统调用**. 通常libc库会将系统调用封装成大家常见的C库函数, 比如malloc(), mmap()等. |
| **内核空间层** | 包含的模块相当丰富, 用户空间和内核空间的接口时系统调用, 因此内核空间层首先需要处理这些内存管理相关的系统调用, 例如sys\_brk, sys\_mmap, sys\_madvise等. 接下来就包括VMA管理, 缺页中断管理, 匿名页面, page cache, 页面回收, 反向映射, slab分配器, 页面管理等模块. |
| **硬件层** | 包含**处理器**的**MMU**, **TLB**和**cache**部件, 以及板载的**物理内存**, 例如LPDDR或者DDR |

# 4 Linux物理内存组织形式

Linux把**物理内存**划分为**三个层次**来管理

| 层次 | 描述 |
|:----|:----|
| **存储节点(Node**) |  CPU被划分为**多个节点(node**), **内存则被分簇**, **每个CPU**对应一个**本地物理内存**, 即**一个CPU\-node**对应一个**内存簇bank**，即**每个内存簇**被认为是**一个节点** |
| **管理区(Zone**)   | **每个物理内存节点node**被划分为**多个内存管理区域**, 用于表示**不同范围的内存**, 内核可以使用**不同的映射方式(！！！**)映射物理内存 |
| **页面(Page**) | 内存被细分为**多个页面帧**, **页面**是**最基本的页面分配的单位**　｜

pg\_data\_t对应一个node，node\_zones包含了不同zone；**zone**下又**定义了per\_cpu\_pageset**，将**page和cpu绑定**。

## 4.1 Node

为了支持**NUMA模型**，也即CPU对不同内存单元的访问时间可能不同，此时系统的物理内存被划分为几个节点(node), 一个node对应一个内存簇bank，即每个内存簇被认为是一个节点

CPU被划分为**结点**, **内存**则被**分簇**, 每个CPU对应一个本地物理内存, 即一个CPU-node对应一个内存簇bank，即每个内存簇被认为是一个节点.

在**分配一个页面**时, Linux采用节点**局部分配的策略**,从最靠近运行中的CPU的节点分配内存,由于**进程往往是在同一个CPU上运行**, 因此**从当前节点**得到的内存很可能被用到.

Linux使用**struct pglist\_data**描述一个**node**(typedef pglist\_data pg\_data\_t)

### 4.1.1 结点的内存管理域

```cpp
typedef struct pglist_data {
	/*  包含了结点中各内存域的数据结构 , 可能的区域类型用zone_type表示*/
    struct zone node_zones[MAX_NR_ZONES];
    /*  指点了备用结点及其内存域的列表，以便在当前结点没有可用空间时，在备用结点分配内存   */
    struct zonelist node_zonelists[MAX_ZONELISTS];
    /*  保存结点中不同内存域的数目    */
    int nr_zones;
} pg_data_t;
```

注意, **当前节点内存域和备用节点内存域用的数据结构不同！！！**

### 4.1.2 结点的内存页面

```cpp
typedef struct pglist_data
{
    /*  指向page实例数组的指针，用于描述结点的所有物理内存页，它包含了结点中所有内存域的页。    */
    struct page *node_mem_map;

	/* 起始页面帧号，指出该节点在全局mem_map中的偏移
    系统中所有的页帧是依次编号的，每个页帧的号码都是全局唯一的（不只是结点内唯一）  */
    unsigned long node_start_pfn;
    /* total number of physical pages 结点中页帧的数目 */
    unsigned long node_present_pages; 
    /*  该结点以页帧为单位计算的长度，包含内存空洞 */
    unsigned long node_spanned_pages; /* total size of physical page range, including holes  */
    /*  全局结点ID，系统中的NUMA结点都从0开始编号  */
    int node_id;		
} pg_data_t;
```

### 4.1.3 交换守护进程

```cpp
typedef struct pglist_data
{
    /*  交换守护进程的等待队列 */
    wait_queue_head_t kswapd_wait;
    wait_queue_head_t pfmemalloc_wait;
    /* 指向负责该结点的交换守护进程的task_struct, 在将页帧换出结点时会唤醒该进程 */
    struct task_struct *kswapd;     /* Protected by  mem_hotplug_begin/end() */
};
```

### 4.1.4 节点状态

| 状态 | 描述 |
|:-----:|:-----:|
| N\_POSSIBLE | 结点在某个时候可能变成联机 |
| N\_ONLINE | 节点是联机的 |
| N\_NORMAL\_MEMORY | 结点是普通内存域 |
| N\_HIGH\_MEMORY | 结点是普通或者高端内存域 |
| **N\_MEMORY** | 结点是普通，高端内存或者MOVEABLE域 |
| N\_CPU | 结点有一个或多个CPU |

其中**N\_POSSIBLE, N\_ONLINE和N\_CPU用于CPU和内存的热插拔**.

对内存管理有必要的标志是N\_HIGH\_MEMORY和N\_NORMAL\_MEMORY

- 如果结点有**普通或高端内存**(**或者！！！**)则使用**N\_HIGH\_MEMORY**
- 仅当结点**没有高端内存**时才设置**N\_NORMAL\_MEMORY**

### 4.1.5 查找内存节点

node\_id作为**全局节点id**。系统中的NUMA结点都是**从0开始编号**的

NUMA系统, 定义了一个**大小为MAX\_NUMNODES类型为pg\_data\_t**的**指针数组node\_data**,数组的大小根据**CONFIG\_NODES\_SHIFT**的配置决定.

对于UMA来说，NODES\_SHIFT为0，所以MAX\_NUMNODES的值为1.  只使用了struct pglist\_data **contig\_page\_data**.

可以使用NODE\_DATA(node\_id)来查找系统中编号为node\_id的结点

```c
[[arch/x86/include/asm/mmzone_64.h]]
extern struct pglist_data *node_data[];
#define NODE_DATA(nid)          (node_data[(nid)])
```

UMA结构下由于只有一个结点, 因此该宏总是返回**全局的contig\_page\_data**.

## 4.2 zone

因为实际的**计算机体系结构**有**硬件的诸多限制**, 这限制了页框可以使用的方式. 尤其是, Linux内核必须处理**80x86体系结构**的**两种硬件约束**.

- **ISA总线的直接内存存储DMA**（**DMA操作！！！**）处理器有一个严格的限制 : 他们**只能对RAM的前16MB进行寻址**

- 在具有大容量RAM的现代**32位计算机(32位才有这个问题！！！**)中, CPU**不能**直接访问**所有的物理地址**, 因为**线性地址空间太小**, 内核不可能直接映射**所有物理内存**到**线性地址空间**.

因此对于内核来说, **不同范围的物理内存**采用**不同的管理方式和映射方式**

Linux使用enum zone\_type来标记内核所支持的所有内存区域

```cpp
enum zone_type
{
#ifdef CONFIG_ZONE_DMA
    ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
    ZONE_DMA32,
#endif
    ZONE_NORMAL,
#ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,
#endif
    ZONE_MOVABLE,
#ifdef CONFIG_ZONE_DEVICE
    ZONE_DEVICE,
#endif
    __MAX_NR_ZONES
};
```

| 管理内存域 | 描述 |
|:---------|:-------|
| ZONE\_DMA | 标记了适合DMA的内存域.该区域的**长度依赖于处理器类型**.这是由于古老的ISA设备强加的边界. 但是为了兼容性,现代的计算机也可能受此影响 |
| ZONE\_DMA32 | 标记了**使用32位地址字可寻址,适合DMA的内存域**(**4GB？？？**).显然,只有在**53位系统中ZONE\_DMA32才和ZONE\_DMA有区别**,在**32位系统中,本区域是空的**, 即长度为0MB,在Alpha和AMD64系统上,该内存的长度可能是从0到4GB |
| ZONE\_NORMAL | 标记了可**直接映射到内存段的普通内存域**.这是在**所有体系结构上保证会存在的唯一内存区域**(**任何体系结构中都会有ZONE\_NORMAL,但是可能是空的!!!**),但**无法保证该地址范围对应了实际的物理地址**(**!!!**). 例如,如果**AMD64系统只有两2G内存**,那么**所有的内存都属于ZONE\_DMA32范围**,而**ZONE\_NORMAL则为空** |
| ZONE\_HIGHMEM | 标记了**超出内核虚拟地址空间**的**物理内存段**,因此这段地址**不能被内核直接映射** |
| ZONE\_MOVABLE | 内核定义了一个伪内存域ZONE\_MOVABLE,在**防止物理内存碎片的机制memory migration中需要使用**该内存域.供防止**物理内存碎片的极致使用** |
| ZONE\_DEVICE | 为支持**热插拔设备**而分配的Non Volatile Memory非易失性内存 |
| MAX\_NR\_ZONES | 充当结束标记, 在内核中想要迭代系统中所有内存域, 会用到该常量 |

根据编译时候的配置, 可能无需考虑某些内存域. 例如**在64位系统**中, 并**不需要高端内存**, ZONE\_HIGHMEM区域总是空的, 可以参见**Documentation/x86/x86\_64/mm.txt**.

一个管理区(zone)由struct zone结构体来描述, 通过typedef被重定义为zone\_t类型

### 4.2.1 高速缓存行

zone数据结构**不需要**像struct page一样**关心数据结构的大小**.

ZONE\_PADDING()让两个自旋锁zone->lock和zone\_lru\_lock这两个很热门的锁可以分布在不同的Cahe Line中, 用空间换取时间, .

内核还对整个zone结构用了\_\_\_\_cacheline\_internodealigned\_in\_smp, 来实现**最优的高速缓存行对齐方式**.

### 4.2.2 水位watermark[NR\_WMARK]与kswapd内核线程

```cpp
struct zone{
    unsigned long watermark[NR_WMARK];
}

enum zone_watermarks
{
        WMARK_MIN,
        WMARK_LOW,
        WMARK_HIGH,
        NR_WMARK
};

#define min_wmark_pages(z) (z->watermark[WMARK_MIN])
#define low_wmark_pages(z) (z->watermark[WMARK_LOW])
#define high_wmark_pages(z) (z->watermark[WMARK_HIGH])
```

| 标准 | 描述 |
|:----:|:---|
| watermark[WMARK\_MIN] | 当空闲页面的数量达到page\_min所标定的数量的时候， 说明页面数非常紧张, **分配页面的动作**和**kswapd线程同步运行**.<br>WMARK\_MIN所表示的**page的数量值**，是在**内存初始化**的过程中调用**free\_area\_init\_core**中计算的。这个数值是根据zone中的page的数量除以一个大于1的系数来确定的。通常是这样初始化的**ZoneSizeInPages/12** |
| watermark[WMARK\_LOW] | 当空闲页面的数量达到WMARK\_LOW所标定的数量的时候，说明页面刚开始紧张, 则**kswapd线程将被唤醒**，并开始释放回收页面 |
| watermark[WMARK\_HIGH] | 当空闲页面的数量达到page\_high所标定的数量的时候， 说明内存页面数充足, 不需要回收, kswapd线程将重新休眠，**通常这个数值是page\_min的3倍** |

kswapd和这3个参数的互动关系如下图：

![config](images/8.jpg)

### 4.2.3 内存域统计信息vm\_stat

```cpp
[include/linux/mmzone.h]
struct zone
{
	  atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
}
```

由enum zone\_stat\_item枚举变量标识

### 4.2.4 zone等待队列表(zone wait queue table)

struct zone中实现了一个**等待队列**,可用于**等待某一页的进程**,内核**将进程排成一个列队**,等待某些条件. 在**条件变成真**时, 内核会**通知进程恢复工作**.

```cpp
struct zone
{
	wait_queue_head_t *wait_table;
	unsigned long      wait_table_hash_nr_entries;
	unsigned long      wait_table_bits;
}
```

| 字段 | 描述 |
|:-----:|:-----:|
| wait\_table | **待一个page释放**的**等待队列哈希表**。它会被wait\_on\_page()，unlock\_page()函数使用. 用**哈希表**，而**不用一个等待队列**的原因，防止进程长期等待资源 |
| wait\_table\_hash\_nr\_entries | **哈希表**中的**等待队列的数量** |
| wait\_table\_bits | **等待队列散列表数组大小**, wait\_table\_size == (1 << wait\_table\_bits)  |

当对**一个page做I/O操作**的时候，I/O操作需要**被锁住**，防止不正确的数据被访问。

**进程在访问page前**，**wait\_on\_page\_locked函数**，使进程**加入一个等待队列**

访问完成后，**unlock\_page函数解锁**其他进程对page的访问。其他正在**等待队列**中的进程**被唤醒**。

- **每个page**都**可以有一个等待队列**，但是太多的**分离的等待队列**使得花费**太多的内存访问周期**。替代的解决方法，就是将**所有的队列**放在**struct zone数据结构**中

- 也可以有一种可能，就是**struct zone中只有一个队列**，但是这就意味着，当**一个page unlock**的时候，访问这个zone里**内存page的所有休眠的进程**(**所有的，而不管是不是访问这个page的进程！！！**)将都**被唤醒**，这样就会出现拥堵（thundering herd）的问题。建立**一个哈希表**管理**多个等待队列**，能解决这个问题，zone->wait\_table就是这个哈希表。哈希表的方法**可能**还是会**造成一些进程不必要的唤醒**。但是这种事情发生的机率不是很频繁的。下面这个图就是进程及等待队列的运行关系：

![config](images/9.jpg)

**等待队列的哈希表的分配和建立**在**free\_area\_init\_core**函数中进行。哈希表的表项的数量在wait\_table\_size() 函数中计算，并且保持在**zone\->wait\_table\_size成员**中。最大**4096个等待队列**。最小是NoPages / PAGES\_PER\_WAITQUEUE的2次方，NoPages是zone管理的**page的数量**，PAGES\_PER\_WAITQUEUE被定义**256**

### 4.2.5 冷热页与Per\-CPU上的页面高速缓存

内核经常**请求和释放单个页框**.为了提升性能,**每个内存管理区**(**zone级别定义！！！**)都定义了一个每CPU(Per\-CPU)的**页面高速缓存**.所有"每CPU高速缓存"包含一些**预先分配的页框**

struct zone的**pageset**成员用于实现**冷热分配器(hot\-cold allocator**)

```cpp
struct zone
{
    struct per_cpu_pageset __percpu *pageset;
};
```
尽管**内存域**可能属于一个**特定的NUMA结点**,因而**关联**到某个**特定的CPU**。但**其他CPU**的**高速缓存**仍然可以包含**该内存域中的页面**（也就是说**CPU的高速缓存可以包含其他CPU的内存域的页！！！**）. 最终的效果是, **每个处理器**都可以访问系统中的**所有页**,尽管速度不同.因而,**特定于内存域的数据结构**不仅要考虑到**所属NUMA结点相关的CPU**, 还必须照顾到系统中**其他的CPU**.

pageset是一个指针, 其容量与系统能够容纳的**CPU的数目的最大值相同**.

### 4.2.6 zonelist内存域存储层次

#### 4.2.6.1 内存域之间的层级结构

**当前结点**与系统中**其他结点**的**内存域**之间存在一种等级次序

我们考虑一个例子, 其中内核想要**分配高端内存**. 

1.	它首先企图在**当前结点的高端内存域**找到一个大小适当的空闲段.如果**失败**,则查看**该结点**的**普通内存域**. 如果还失败, 则试图在**该结点**的**DMA内存域**执行分配.

2.	如果在**3个本地内存域**都无法找到空闲内存, 则查看**其他结点**. 在这种情况下, **备选结点**应该**尽可能靠近主结点**, 以最小化由于访问非本地内存引起的性能损失.

内核定义了内存的一个**层次结构**, 首先试图分配"廉价的"内存. 如果失败, 则根据访问速度和容量, 逐渐尝试分配"更昂贵的"内存.

**高端内存是最廉价的**, 因为内核**没有**任何部分**依赖**于从该内存域分配的内存. 如果**高端内存域用尽**, 对内核**没有任何副作用**, 这也是优先分配高端内存的原因.

**其次是普通内存域**, 这种情况有所不同. **许多内核数据结构必须保存在该内存域**, 而不能放置到高端内存域.

因此如果普通内存完全用尽, 那么内核会面临紧急情况. 所以只要高端内存域的内存没有用尽, 都不会从普通内存域分配内存.

**最昂贵的是DMA内存域**, 因为它用于外设和系统之间的数据传输. 因此从该内存域分配内存是最后一招.

#### 4.2.6.2 备用节点内存域zonelist结构

内核还针对**当前内存结点**的**备选结点**,定义了一个**等级次序**.这有助于在**当前结点所有内存域**的内存都**用尽**时, 确定一个**备选结点**

内核使用pg\_data\_t中的**zonelist数组**, 来**表示所描述的层次结构**.

```cpp
typedef struct pglist_data {
	struct zonelist node_zonelists[MAX_ZONELISTS];
	/*  ......  */
}pg_data_t;
```

node\_zonelists**数组**对每种可能的**内存域类型**, 都配置了一个**独立的数组项**.

```cpp
enum
{
    ZONELIST_FALLBACK,      /* zonelist with fallback */
#ifdef CONFIG_NUMA
    ZONELIST_NOFALLBACK,
#endif
    MAX_ZONELISTS
};
```

**UMA结构**下, 数组大小**MAX\_ZONELISTS = 1**, NUMA下需要多余的**ZONELIST\_NOFALLBACK**用以表示**当前结点**的信息

由于该**备用列表**必须包括**所有结点**的**所有内存域(！！！**)，因此**由MAX\_NUMNODES * MAX\_NZ\_ZONES项**组成，外加一个**用于标记列表结束的空指针**

```cpp
struct zonelist {
    struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};

struct zoneref {
    struct zone *zone;      /* Pointer to actual zone */
    int zone_idx;       /* zone_idx(zoneref->zone) */
};
```

#### 4.2.6.3 内存域的排列方式

NUMA系统中存在**多个节点**, **每个结点**中可以包含**多个zone**.

- Legacy方式, **每个节点只排列自己的zone**；

![Legacy方式](./images/10.jpg)

- Node方式, 按**节点顺序**依次排列，先排列本地节点的所有zone，再排列其它节点的所有zone。

![Node方式](./images/11.jpg)

- Zone方式, 按**zone类型**从高到低依次排列各节点的相同类型zone

![Zone方式](./images/12.jpg)

可通过启动参数"**numa\_zonelist\_order**"来配置**zonelist order**，内核定义了3种配置

```cpp
#define ZONELIST_ORDER_DEFAULT  0 /* 智能选择Node或Zone方式 */
#define ZONELIST_ORDER_NODE     1 /* 对应Node方式 */
#define ZONELIST_ORDER_ZONE     2 /* 对应Zone方式 */
```

## 4.3 Page

**页帧(page frame**)代表了系统内存的最小单位, 对内存中的**每个页**都会创建一个struct page的一个实例. 内核必须要**保证page结构体足够的小**. 每一页物理内存叫页帧，以页为单位对内存进行编号，该编号可作为页数组的索引，又称为页帧号.

### 4.3.1 mapping & index

**mapping**指定了**页帧**（**物理页！！！**）所在的**地址空间**,**index**是**页帧**在映射的**虚拟空间内部**的偏移量.**地址空间**是一个非常一般的概念.例如,可以用在**向内存读取文件**时. 地址空间用于将**文件的内容**与装载**数据的内存区关联起来**.

mapping不仅能够保存一个指针,而且还能包含一些额外的信息,用于判断页是否属于**未关联到地址空间**的**某个匿名内存区**.

page\->mapping本身是一个**指针**，指针地址的**低几个bit**因为**对齐的原因**都是**无用的bit**，内核就根据这个特性利用这几个bit来让page->mapping实现更多的含义。**一个指针多个用途**，这个也是内核为了**减少page结构大小**的办法之一。目前用到**最低2个bit位**。

```c
#define PAGE_MAPPING_ANON	0x1
#define PAGE_MAPPING_MOVABLE	0x2
#define PAGE_MAPPING_KSM	(PAGE_MAPPING_ANON | PAGE_MAPPING_MOVABLE)
#define PAGE_MAPPING_FLAGS	(PAGE_MAPPING_ANON | PAGE_MAPPING_MOVABLE)
```

1. 如果page->mapping == NULL，说明该**page**属于**交换高速缓存页**（**swap cache**）；当需要使用地址空间时会指定**交换分区的地址空间**swapper\_space。

2. 如果page->mapping != NULL，第0位bit[0] = 0，说明该page属于**页缓存**或**文件映射**，mapping指向**文件的地址空间**address\_space。

3. 如果page->mapping != NULL，第0位bit[0] != 0，说明该page为**匿名映射**，mapping指向**struct anon\_vma对象**。

### 4.3.2 lru链表头

最近、最久未使用**struct slab结构指针**变量

lru：链表头，主要有3个用途：

1.	则**page处于伙伴系统**中时，用于链接**相同阶的伙伴**（只使用伙伴中的**第一个page的lru**即可达到目的）。

2.	**设置PG\_slab**, 则**page属于slab**，page\->lru.next指向page驻留的的缓存的管理结构，page->lru.prec指向保存该page的slab的管理结构。

3.	page被**用户态使用**或被当做**页缓存使用**时，用于将该**page**连入zone中**相应的lru链表**，供**内存回收**时使用。

### 4.3.3 内存页标识pageflags

### 4.3.4 全局页面数组mem\_map

mem\_map是一个**struct page的数组**，管理着系统中**所有的物理内存页面**。在系统启动的过程中，创建和分配mem\_map的内存区域.

UMA体系结构中，**free\_area\_init**函数在系统唯一的struct node对象**contig\_page\_data**中**node\_mem\_map**成员赋值给全局的mem\_map变量

# 5 物理内存初始化

