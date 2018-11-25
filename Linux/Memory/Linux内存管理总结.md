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

内核经常**请求和释放单个页框(！！！**).为了提升性能,**每个内存管理区**(**zone级别定义！！！**)都定义了一个每CPU(Per\-CPU)的**页面高速缓存**. 所以"**每CPU高速缓存**"包含一些**预先分配的页框**

struct zone的**pageset**成员用于实现**冷热分配器(hot\-cold allocator**)

```cpp
struct zone
{
    struct per_cpu_pageset __percpu *pageset;
};
```

**页面是热的**，意味着**页面**已经**加载到CPU的高速缓存**,与在内存中的页相比,其数据**访问速度更快**. 相反,**冷页**则**不在高速缓存**中.在多处理器系统上**每个CPU**都有**一个或者多个高速缓存**.各个CPU的管理必须是独立的.

尽管**内存域**可能属于一个**特定的NUMA结点**,因而**关联**到某个**特定的CPU**。但**其他CPU**的**高速缓存**仍然可以包含**该内存域中的页面**（也就是说**CPU的高速缓存可以包含其他CPU的内存域的页！！！**）. 最终的效果是, **每个处理器**都可以访问系统中的**所有页**,尽管速度不同.因而,**特定于内存域的数据结构**不仅要考虑到**所属NUMA结点相关的CPU**, 还必须照顾到系统中**其他的CPU**.

**pageset是一个指针**, 其容量与系统能够容纳的**CPU的数目的最大值相同(！！！**).

数组元素类型为**per\_cpu\_pageset**

```cpp
struct per_cpu_pageset {
       struct per_cpu_pages pcp;
#ifdef CONFIG_NUMA
       s8 expire;
#endif
#ifdef CONFIG_SMP
       s8 stat_threshold;
       s8 vm_stat_diff[NR_VM_ZONE_STAT_ITEMS];
#endif
};
```

该结构由一个**per\_cpu\_pages pcp变量**组成, 该数据结构定义如下

```cpp
struct per_cpu_pages {
	int count;              /* number of pages in the list 列表中的页数  */
	int high;               /* high watermark, emptying needed 页数上限水印, 在需要的情况清空列表  */
    int batch;              /* chunk size for buddy add/remove,  添加/删除多页块的时候, 块的大小  */

	/* Lists of pages, one per migrate type stored on the pcp-lists 页的链表*/
       struct list_head lists[MIGRATE_PCPTYPES];
};
```

| 字段 | 描述 |
|:---:|:----:|
| count | 记录了与该列表相关的**页的数目** |
| high  | 是一个**水位**. 如果**count的值超过了high**, 则表明列表中的**页太多**了 |
| batch | 如果可能, **CPU的高速缓存**不是用**单个页**来填充的,而是**多个页组成的块**,batch作为每次添加/删除页时多个页组成的**块大小**的一个参考值 |
| list | 一个**双链表**, 保存了**当前CPU**的**冷页或热页**, 可使用内核的标准方法处理 |

在内核中**只有一个子系统**会积极的尝试为任何对象**维护per\-cpu上的list链表**,这个子系统就是**slab分配器**.

- struct per\_cpu\_pageset具有一个字段, 该字段

- struct per\_cpu\_pages则维护了链表中目前已有的一系列页面, 高极值和低极值决定了何时填充该集合或者释放一批页面, 变量决定了**一个块**中应该分配**多少个页面**, 并最后决定在页面前的实际链表中分配多少个页面

**per\_cpu\_pageset结构**是内核的**各个zone**用于**每CPU**的**页面高速缓存管理结构**。该**高速缓存**包含一些**预先分配的页面**，以用于满足**本地CPU**发出的**单一内存页请求**。而struct per\_cpu_pages定义的**pcp**是该**管理结构的成员**，用于**具体页面管理**。这是**一个队列(其实是3个迁移类型每个一个队列**), 统一管理冷热页，**热页面在队列前面**，而**冷页面则在队列后面**, 这个从释放页面也能看出来。

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

**UMA结构**下, 数组大小**MAX\_ZONELISTS = 1**

NUMA下需要多余的**ZONELIST\_NOFALLBACK**用以表示**当前结点(！！！**)的信息

由于该**备用列表**必须包括**所有结点(！！！包括当前节点！！！**)的**所有内存域(！！！**)，因此**由MAX\_NUMNODES * MAX\_NZ\_ZONES项**组成，外加一个**用于标记列表结束的空指针**

```cpp
struct zonelist {
    struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};

struct zoneref {
    struct zone *zone;      /* Pointer to actual zone */
    int zone_idx;       /* zone_idx(zoneref->zone) */
};
```

也就是说, 每个节点node有一个zonelist数组(对于UMA只有**一个数组项**, NUMA有**两个数组项**), 每个数组项是一个zonelist的数据结构, 该数据结构由**zone信息组成的数组**(备用列表包含**所有节点的所有内存域**, 所以**数组大小**是MAX\_NUMNODES \* MAX\_NZ\_ZONES + 1, 包含一个**标记结束的空指针**), 这个数组项中包含zone结构和索引

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

#### 4.2.6.4 build\_all\_zonelists初始化内存节点

内核通过build\_all\_zonelists初始化了内存结点的zonelists域

- 首先内核通过set\_zonelist\_order函数设置了zonelist\_order,如下所示, 参见mm/page_alloc.c?v=4.7, line 5031

- 建立备用层次结构的任务委托给build\_zonelists,该函数为每个NUMA结点都创建了相应的数据结构. 它需要指向相关的pg\_data\_t实例的指针作为参数

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

# 5 Linux分页机制

分页的基本方法是将**地址空间**人为地等分成**某一个固定大小的页**；每一**页大小**由**硬件来决定**，或者是由**操作系统来决定**(如果**硬件支持多种大小的页**)。目前，以大小为4KB的分页是绝大多数PC操作系统的选择。

- **逻辑空间（线性空间！！！**）等分为**页**；并**从0开始编号**
- **内存空间（物理空间！！！**）等分为**块**，**与页面大小相同**；**从0开始编号**
- **分配内存**时，**以块为单位**将进程中的**若干个页**分别装入内存空间

关于进程分页。当我们把进程的**虚拟地址空间按页**来分割，**常用的数据和代码**会被装在到**内存**；**暂时没用到的是数据和代码**则保存在**磁盘**中，需要用到的时候，再从磁盘中加载到内存中即可。

这里需要了解三个概念：

1. **虚拟页**(VP, Virtual Page), **虚拟空间**中的**页**；

2. **物理页**(PP, Physical Page), **物理内存**中的**页**；

3. **磁盘页**(DP, Disk Page), **磁盘**中的**页**。

**虚拟内存**的实现需要硬件的支持，从Virtual Address到Physical Address的映射，通过一个叫**MMU**(**Memory Mangement Unit**)的部件来完成

**分页单元(paging unit**)把**线性地址**转换成**物理地址**。其中的一个关键任务就是把**所请求的访问类型**与**线性地址的访问权限相比较**，如果这次内存访问是**无效**的，就产生一个**缺页异常**。
 
- **页**：为了更高效和更经济的管理内存，**线性地址**被分为以固定长度为单位的组，成为**页**。页内部**连续的线性地址空间（页是连续的！！！**）被映射到**连续的物理地址**中。这样，内核可以指定**一个页（！！！**）的物理地址和对应的**存取权限**，而不用指定全部线性地址的存取权限。这里说**页**，同时指**一组线性地址**以及这组地址**包含的数据**

- **页框**：分页单元把所有的**RAM**分成固定长度的**页框**(page frame)(有时叫做**物理页**)。每一个页框包含一个页(page)，也就是说一个页框的长度与一个页的长度一致。页框是主存的一部分，因此也是一个存储区域。区分**一页**和**一个页框**是很重要的，前者只是**一个数据块**，可以存放在**任何页框**或**磁盘（！！！线性空间的数据在物理页帧或磁盘可以是任何位置！！！**）中。

- **页表**：把**线性地址**映射到**物理地址**的数据结构称为**页表**(page table)。页表存放在**主存**中，并在启用分页单元之前必须由内核对页表进行适当的初始化。

关于硬件分页机制看其他资料

## 5.1 Linux中的分页层次

目前的内核的内存管理总是**固定使用四级页表**, 而不管底层处理器是否如此.

| 单元 | 描述 |
|:---:|:----:|
| 页全局目录 | Page Global Directory  |
| 页上级目录	| Page Upper Directory  |
| 页中间目录	| Page Middle Directory |
| 页表	        | Page Table 			|
| 页内偏移      | Page Offset		    |

- **页全局目录**包含若干**页上级目录**的地址；
- **页上级目录**又依次包含若干**页中间目录**的地址；
- 而**页中间目录**又包含若干****页表****的地址；
- 每一个**页表项**指向一个**页框**。

**Linux页表管理**分为两个部分,第一个部分**依赖于体系结构**,第二个部分是**体系结构无关的**.

所有**数据结构**几乎都定义在**特定体系结构的文件**中.这些数据结构的定义可以在**头文件arch/对应体系/include/asm/page.h**和**arch/对应体系/include/asm/pgtable.h**中找到. 但是对于**AMD64**和**IA\-32**已经统一为**一个体系结构**.但是在处理页表方面仍然有很多的区别,因为相关的定义分为**两个不同的文件**arch/x86/include/asm/page\_32.h和arch/x86/include/asm/page\_64.h, 类似的也有pgtable\_xx.h.


对于不同的体系结构，Linux采用的**四级页表目录**的大小有所不同(**页大小**都是**4KB**情况下）：

- 对于**i386**而言，仅采用**二级页表**，即**页上层目录**和**页中层目录**长度为0；**10(PGD)\+0(PUD)\+0(PMD)\+10(PTE)\+12(offset)**。
- 对于启用**PAE**的**i386**，采用了**三级页表**，即**页上层目录**长度为0；**2(PGD)\+0(PUD)\+9(PMD)\+9(PTE)\+12(offset)**。
- 对于**64位**体系结构，可以采用三级或四级页表，具体选择由**硬件决定**。**x86\_64下，9(PGD)\+9(PUD)\+9(PMD)\+9(PTE)\+12(offset)**。

对于**没有启用物理地址扩展(PAE**)的**32位系统**，**两级页表**已经足够了。从本质上说Linux通过使“**页上级目录**”位和“**页中间目录**”位**全为0（！！！**），彻底取消了页上级目录和页中间目录字段。不过，页上级目录和页中间目录在**指针序列中**的位置被**保留**，以便同样的代码在32位系统和64位系统下都能使用。内核为**页上级目录**和**页中间目录**保留了一个位置，这是通过把它们的**页目录项数**设置为**1**，并把这**两个目录项**映射到**页全局目录**的一个合适的**目录项**而实现的。

## 5.2 Linux中分页相关数据结构

Linux分别采用**pgd\_t、pud\_t、pmd\_t和pte\_t**四种数据结构来表示**页全局目录项、页上级目录项、页中间目录项和页表项**。这四种数据结构本质上都是**无符号长整型unsigned long**, 这些都是**表项！！！不是表本身！！！**

linux中使用下列宏简化了页表处理，对于**每一级页表**都使用有以下三个关键描述宏：

| 宏字段| 描述 |
| ------------- |:-------------|
| XXX\_SHIFT| 指定**一个相应级别页表项(！！！)可以映射的区域大小的位数** |
| XXX\_SIZE| **一个相应级别页表项(！！！)可以映射的区域大小** |
| XXX\_MASK| 用以**屏蔽一个相应级别页表项可以映射的区域大小的所有位数**。 |

我们的**四级页表**，对应的宏前缀分别由**PAGE，PMD，PUD，PGDIR**

### 5.2.1 PAGE宏--页表(Page Table)

```
 #define PAGE_SHIFT      12
 #define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
 #define PAGE_MASK       (~(PAGE_SIZE-1))
```

- 当用于80x86处理器时，**PAGE\_SHIFT**返回的值为**12（这个值是写死的！！！**），**页表项**中存放的是**一个页的基地址**，所以**页表项能映射的区域大小的位数自然就取决于页的位数！！！**。**一个页表项**可以映射的**区域大小的位数**.

- 由于**页内所有地址都必须放在offset字段**，因此80x86系统的页的大小**PAGE\_SIZE**是**2\^{12}**=**4096字节**。一个页表项可以映射的区域的大小.

- PAGE\_MASK宏产生的值为0xfffff000，用以**屏蔽Offset字段的所有位（低12位全为0**）。屏蔽**一个页表项可以映射的区域大小**的**所有位数**。

### 5.2.2 PMD\-Page Middle Directory (页目录)

| 字段| 描述 |
| ------------- |:-------------|
| PMD\_SHIFT| 指定**线性地址的Offset和Table字段**的总位数；换句话说，是**页中间目录项**可以映射的**区域大小的位数** |
| PMD\_SIZE| 用于计算由页中间目录的**一个单独表项**所映射的区域大小，也就是一个**页表的大小** |
| PMD\_MASK| 用于**屏蔽Offset字段与Table字段的所有位** |

**页中间目录项**存放的是**一个页表的基地址**，所以**一个页中间目录项所能映射的区域大小位数**自然取决于(**页表项位数) \+ (页大小位数)！！！**

**当PAE被禁用时**，32\-bit分页的情况下(10 \+ 10 \+12)

- PMD\_SHIFT产生的值为**22**（来自**Offset的12位**加上来自**Table<页表>的10位**）
- PMD\_SIZE产生的值为2\^{22}或**4MB**
- PMD\_MASK产生的值为**0xffc00000（低22位为0**）。

相反，当PAE被激活时（2\+9\+9\+12或IA\-32e的2\+9\+9\+9\+12**），

- PMD\_SHIFT产生的值为**21** (来自**Offset的12**位加上来自**Table的9**位)
- PMD\_SIZE产生的值为**2\^{21}**或**2MB**
- PMD\_MASK产生的值为**0xffe00000**。

**大型页（无论32-bit分页的4MB页、PAE分页的2MB页，IA-32e分页的2MB页！！！Linux是支持这个的！！！**）不使用**最后一级页表**，所以产生大型页尺寸的**LARGE\_PAGE\_SIZE宏**等于**PMD\_SIZE**(**2\^{PMD\_SHIFT}**)，而在大型页地址中用于**屏蔽Offset字段和Table字段的所有位**的**LARGE\_PAGE\_MASK**宏，就等于**PMD\_MASK**。

### 5.2.3 PUD\_SHIFT-页上级目录(Page Upper Directory)

**页上级目录项**存放的是**一个页中间目录的基地址**，所以**一个页上级目录项所能映射的区域大小自然取决于2的（页中间目录项\+页表项\+页大小）位数的平方！！！**。

对于i386而言，仅采用二级页表，即**页上层目录**和**页中层目录长度**为**0**，所以**PUD\_SHIFT总是等价于PMD\_SHIFT(22=10<PTE>\+12<offset**>)，而**PUD\_SIZE**则等于**4MB**。

对于启用PAE的i386，采用了三级页表，即**页上层目录**长度为**0**,页中层目录长度为9，所以**PUD\_SHIFT总是等价于30（=9<PMD>\+9<PTE>\+12<offset**>)，而**PUD\_SIZE**则等于**1GB**

### 5.2.4 PGDIR\_SHIFT\-页全局目录(Page Global Directory)

| 字段| 描述 |
| ------------- |:-------------|
| PGDIR\_SHIFT| 确定**一个全局页目录项**能映射的区域大小的位数 |
| PGDIR\_SIZE| 用于计算页全局目录中一个单独表项所能映射区域的大小 |
| PGDIR\_MASK| 用于屏蔽Offset, Table，Middle Air及Upper Air的所有位 |

**当PAE 被禁止时**，

- PGDIR\_SHIFT 产生的值为**22**（与PMD\_SHIFT 和PUD\_SHIFT 产生的值相同），
- PGDIR\_SIZE 产生的值为 2\^22 或 4 MB，
- PGDIR\_MASK 产生的值为 0xffc00000。

相反，**当PAE被激活时**，

- PGDIR\_SHIFT 产生的值为**30** (**12位Offset** 加**9位Table**再加**9位 Middle** Air), 
- PGDIR\_SIZE 产生的值为2\^30 或 1 GB
- PGDIR\_MASK产生的值为0xc0000000

## 5.3 页表处理函数

内核还提供了许多宏和函数用于**读或修改页表表项**：

.......

查询页表项中任意一个标志的当前值

### 5.3.1 简化页表项的创建和撤消

当使用**两级页表**时，创建或删除一个**页中间目录项**是不重要的。如本节前部分所述，**页中间目录仅含有一个指向下属页表的目录项**。所以，**页中间目录项**只是**页全局目录中的一项**而已。然而当处理页表时，**创建一个页表项**可能很复杂，因为包含页表项的那个页表可能就不存在。在这样的情况下，有必要**分配一个新页框，把它填写为0，并把这个表项加入**。

如果**PAE**被激活，内核使用**三级页表**。当内核创建一个新的**页全局目录**时，同时也**分配四个相应的页中间目录**；只有当**父页全局目录被释放**时，这**四个页中间目录才得以释放**。当使用**两级**或**三级**分页时，**页上级目录项**总是被映射为**页全局目录**中的**一个单独项**。与以往一样，下表中列出的函数描述是针对80x86构架的。

| 函数名称 | 说明 |
| ------------- |:-------------|
| pgd\_alloc(mm) | 分配一个**新的页全局目录**。如果**PAE被激活**，它还分配**三个对应用户态线性地址**的**子页中间目录**。参数mm(内存描述符的地址)在80x86 构架上被忽略 |
| pgd\_free( pgd) | 释放页全局目录中地址为 pgd 的项。如果 PAE 被激活，它还将释放用户态线性地址对应的三个页中间目录 |
| pud\_alloc(mm, pgd, addr) | 在两级或三级分页系统下，这个函数什么也不做：它仅仅返回页全局目录项 pgd 的线性地址 |
| pud\_free(x) | 在**两级或三级分页系统**下，这个宏什么也不做 |
| pmd_alloc(mm, pud, addr) | 定义这个函数以使普通三级分页系统可以为线性地址 addr 分配一个新的页中间目录。如果 PAE 未被激活，这个函数只是返回输入参数 pud 的值，也就是说，返回页全局目录中目录项的地址。如果 PAE 被激活，该函数返回线性地址 addr 对应的页中间目录项的线性地址。参数 mm 被忽略 |
| pmd\_free(x) | 该函数什么也不做，因为页中间目录的分配和释放是随同它们的父全局目录一同进行的 |
| pte\_alloc\_map(mm, pmd, addr) | 接收页中间目录项的地址 pmd 和线性地址 addr 作为参数，并返回与 addr 对应的页表项的地址。如果页中间目录项为空，该函数通过调用函数 pte\_alloc\_one( ) 分配一个新页表。如果分配了一个新页表， addr 对应的项就被创建，同时 User/Supervisor 标志被设置为 1 。如果页表被保存在高端内存，则内核建立一个临时内核映射，并用 pte\_unmap 对它进行释放 |
| pte\_alloc\_kernel(mm, pmd, addr) | 如果与地址 addr 相关的页中间目录项 pmd 为空，该函数分配一个新页表。然后返回与 addr 相关的页表项的线性地址。该函数仅被主内核页表使用 |
| pte\_free(pte) | 释放与页描述符指针 pte 相关的页表 |
| pte\_free\_kernel(pte) | 等价于 pte\_free( ) ，但由主内核页表使用 |
| clear\_page\_range(mmu, start,end) | 从线性地址 start 到 end 通过反复释放页表和清除页中间目录项来清除进程页表的内容 |

## 5.4 线性地址转换

地址转换过程有了上述的基本知识，就很好理解四级页表模式下如何将虚拟地址转化为逻辑地址了。基本过程如下：

- 1.从**CR3寄存器**中读取**页目录所在物理页面的基址**(即所谓的页目录基址)，从线性地址的第一部分获取页目录项的索引，两者相加得到**页目录项的物理地址(！！！**）。

- 2.**第一次读取内存**得到**pgd\_t结构的目录项**，从中取出物理页基址取出(具体位数与平台相关，如果是32系统，则为20位)，即页上级页目录的物理基地址。

- 3.从线性地址的第二部分中取出页上级目录项的索引，与页上级目录基地址相加得到**页上级目录项的物理地址（！！！**）。

- 4.**第二次读取内存**得到**pud\_t结构的目录项**，从中取出页中间目录的物理基地址。

- 5.从线性地址的第三部分中取出页中间目录项的索引，与页中间目录基址相加得到**页中间目录项的物理地址（！！！**）。

- 6.**第三次读取内存**得到**pmd\_t结构的目录项**，从中取出页表的物理基地址。

- 7.从线性地址的第四部分中取出页表项的索引，与页表基址相加得到**页表项的物理地址（！！！**）。

- 8.**第四次读取内存**得到**pte\_t结构的目录项**，从中取出物理页的基地址。

- 9.从线性地址的第五部分中取出物理页内偏移量，与物理页基址相加得到**最终的物理地址（！！！**）。

- 10.**第五次读取内存**得到最终要访问的数据。

整个过程是比较机械的，每次转换先获取物理页基地址，再从线性地址中获取索引，合成物理地址后再访问内存。不管是**页表**还是要访问的**数据**都是**以页为单位存放在主存中**的，因此每次访问内存时都要先获得基址，再通过索引(或偏移)在页内访问数据，因此可以将线性地址看作是若干个索引的集合。

![config](images/13.png)

![config](images/14.png)

## 5.5 Linux中通过4级页表访问物理内存

linux中**每个进程**有它自己的**PGD**( Page Global Directory)，它是**一个物理页（一个物理页！！！**），并包含一个**pgd\_t数组**, pgd\_t是**表项**, 所以这里是**pgd entry数组, 即表项数组！！！**。

**进程的pgd\_t**数据见task\_struct \-\> mm\_struct \-\> pgd\_t \* pgd;

通过如下如下几个函数, **不断向下索引**, 就可以**从进程的页表**中搜索**特定地址**对应的**页面对象**

| 宏函数| 说明 |
| ------------- |:-------------|
| **pgd\_offset**  | 根据**当前虚拟地址**和**当前进程的mm\_struct**获取**pgd项** |
| **pud\_offset** | 参数为指向**页全局目录项的指针pgd**和**线性地址addr** 。这个宏产生**页上级目录中目录项addr对应的线性地址**。在两级或三级分页系统中，该宏产生 pgd ，即一个页全局目录项的地址 |
| pmd\_offset | 根据通过**pgd\_offset获取的pgd 项**和**虚拟地址**，获取相关的**pmd项(即pte表的起始地址！！!**) |
| **pte\_offset** | 根据通过pmd\_offset获取的**pmd项**和**虚拟地址**，获取**相关的pte项(即物理页的起始地址！！！**) |

根据**虚拟地址**获取**物理页**的示例代码详见mm/memory.c中的函数**follow\_page\_mask**

# 6 内存布局探测

**linux**在**被bootloader加载到内存**后， cpu**最初执行**的内核代码是**arch/x86/boot/header.S**汇编文件中的\_**start例程**，设置好**头部header**，其中包括**大量的bootloader参数**。接着是其中的**start\_of\_setup例程**，这个例程在做了一些**准备工作**后会通过**call main**跳转到**arch/x86/boot/main.c:main()函数**处执行，这就是众所周知的x86下的main函数，它们都工作在**实模式**下。这里面能第一次看到与内存管理相关代码, 这就是调用detect\_memory()检测系统物理内存.

作为**内核的内存布局来源**，BIOS提供了两套内存布局信息，第一个是**legacy启动**时提供的**e820 memory map**，另一个是**efi启动**时提供的**efi memory map**。

**下面内容针对legacy启动**。

```
main()                      #/arch/x86/boot/main.c

+——> detect_memory()        #/arch/x86/boot/main.c

+——>detect_memory_e820()    #/arch/x86/boot/memory.c
```

**循环调用BIOS的0x15中断**, ax赋值为0xe820， 将会返回被BIOS保留内存地址范围以及系统可以使用的内存地址范围。所有通过中断获取的数据将会填充在boot\_params.e820\_map中.

由于历史原因，一些**I/O设备**也会占据一部分**内存物理地址空间**，因此**系统**可以使用的**物理内存空间**是**不连续**的，系统内存被分成了**很多段**，**每个段**的**属性也是不一样**的。**BIOS的int 0x15中断**查询物理内存时**每次返回一个内存段的信息**，因此要想返回系统中所有的物理内存，我们必须以**迭代的方式**去查询。

detect\_memory\_e820()函数把int 0x15放到一个do\-while循环里，每次得到的一个**内存段**放到**struct e820entry**里，而struct e820entry的结构正是e820返回结果的结构。像其它启动时获得的结果一样，最终都会**被放到boot\_params**里，探测到的各个内存段情况被放到了boot\_params.e820\_map。

这里存放**中断返回值的e820entry**结构，以及表示内存图的e820map结构均位于arch/x86/include/asm/e820.h中，如下：

```c
struct e820entry {
    __u64 addr; /* 内存段的开始 */
    __u64 size; /* 内存段的大小 */
    __u32 type; /* 内存段的类型 */
} __attribute__((packed));

struct e820map {
	__u32 nr_map;
	struct e820entry map[E820_X_MAX];
};
```

内存探测用于检测出系统有多少个通常不连续的内存区块。之后要建立一个描述这些内存块的内存图数据结构，这就是上面的e820map结构，其中nr\_map为检测到的系统中内存区块数，不能超过E820\_X\_MAX（定义为128），**map数组**描述各个内存块的情况，包括其开始地址、内存块大小、类型。

这是在**实模式**下完成的内存布局探测，此时**尚未进入保护模式**。

# 5 内存管理的三个阶段

linux内核的**内存管理分三个阶段**。

| 阶段 | 起点 | 终点 | 描述 |
|:-----|:-----|:-----|:-----|
| 第一阶段 | 系统启动 | bootmem或者memblock初始化完成 | 此阶段只能使用**memblock\_reserve函数**分配内存，**早期内核**中使用init\_bootmem\_done = 1标识此阶段结束 |
| 第二阶段 | bootmem或者memblock初始化完 | buddy完成前 | **引导内存分配器bootmem**或者**memblock**接受内存的管理工作, **早期内核**中使用mem\_init\_done = 1标记此阶段的结束 |
| 第三阶段 | buddy初始化完成 | 系统停止运行 | 可以用**cache和buddy分配**内存 |

# 6 系统初始化过程中的内存管理

对于32位的系统，通过调用链arch/x86/boot/main.c:main()--->arch/x86/boot/pm.c:go\_to\_protected\_mode()--->arch/x86/boot/pmjump.S:protected\_mode\_jump()--->arch/i386/boot/compressed/head\_32.S:startup\_32()--->arch/x86/kernel/head\_32.S:startup\_32()--->arch/x86/kernel/head32.c:i386\_start\_kernel()--->init/main.c:start\_kernel()，到达众所周知的**Linux内核启动函数start\_kernel**()

先看start\_kernel如何**初始化系统**的.

截取与内存管理相关部分.

```c
asmlinkage __visible void __init start_kernel(void)
{

    /*  设置特定架构的信息
     *	同时初始化memblock  */
    setup_arch(&command_line);
    mm_init_cpumask(&init_mm);
    setup_per_cpu_areas();
	/*  初始化内存结点和内段区域  */
    build_all_zonelists(NULL, NULL);
    page_alloc_init();

    /*
     * These use large bootmem allocations and must precede
     * mem_init();
     * kmem_cache_init();
     */
    mm_init();
    kmem_cache_init_late();
	kmemleak_init();
    setup_per_cpu_pageset();
    rest_init();
}
```

| 函数  | 功能 |
|:----|:----|
| setup\_arch | 是一个**特定于体系结构**的设置函数, 其中一项任务是负责**初始化自举分配器** |
| mm\_init\_cpumask | 初始化**CPU屏蔽字** |
| **setup\_per\_cpu\_areas** | 函数给**每个CPU分配内存**，并**拷贝.data.percpu段**的数据.为系统中的**每个CPU的per\_cpu变量申请空间**. 在**SMP系统**中, setup\_per\_cpu\_areas初始化源代码中(使用**per\_cpu宏**)定义的**静态per\-cpu变量**, 这种变量对系统中**每个CPU都有一个独立的副本**. 此类变量保存在**内核二进制影像**的一个**独立的段**中, setup\_per\_cpu\_areas的目的就是为系统中**各个CPU分别创建一份这些数据的副本**, 在**非SMP系统**中这是一个**空操作** |
| build\_all\_zonelists | 建立并初始化**结点**和**内存域**的数据结构 |
| **mm\_init** | 建立了内核的**内存分配器(！！！**), 其中通过**mem\_init停用bootmem**分配器并迁移到**实际的内存管理器(比如伙伴系统**), 然后调用**kmem\_cache\_init**函数初始化内核内部**用于小块内存区的分配器** |
| kmem\_cache\_init\_late | 在**kmem\_cache\_init**之后,完善分配器的**缓存机制**,　当前3个可用的内核内存分配器**slab**, **slob**, **slub**都会定义此函数　|
| kmemleak\_init | Kmemleak工作于内核态，Kmemleak提供了一种**可选的内核泄漏检测**，其方法类似于**跟踪内存收集器**。当独立的对象没有被释放时，其报告记录在/sys/kernel/debug/kmemleak中, Kmemcheck能够帮助定位大多数内存错误的上下文 |
| setup\_per\_cpu\_pageset | **初始化CPU高速缓存行**, 为**pagesets**的**第一个数组元素分配内存**, 换句话说, 其实就是**第一个系统处理器分配**. 由于在分页情况下，**每次存储器访问都要存取多级页表**，这就大大降低了访问速度。所以，为了提高速度，在**CPU中**设置一个**最近存取页面**的**高速缓存硬件机制**，当进行**存储器访问**时，**先检查**要访问的**页面是否在高速缓存(！！！**)中. |

# 7 特定于体系结构的内存初始化工作

setup\_arch()完成与体系结构相关的一系列初始化工作，其中就包括各种内存的初始化工作，如内存图的建立、管理区的初始化等等。对x86体系结构，setup\_arch()函数在arch/x86/kernel/setup.c中，如下：

```c
void __init setup_arch(char **cmdline_p)
{
	/* ...... */
	x86_init.oem.arch_setup();
	/* 建立内存图 */
	setup_memory_map(); 
	parse_setup_data();

    /* 找出最大可用内存页面帧号 */
	max_pfn = e820_end_of_ram_pfn();

	/* update e820 for memory not covered by WB MTRRs */
	mtrr_bp_init();
	if (mtrr_trim_uncached_memory(max_pfn))
		max_pfn = e820_end_of_ram_pfn();

#ifdef CONFIG_X86_32
	/* max_low_pfn get updated here */
	/* max_low_pfn在这里更新 */
	/* 找出低端内存的最大页帧号 */
	find_low_pfn_range();
#else
	check_x2apic();
    // 非32位获取max_low_pfn
	if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
		max_low_pfn = e820_end_of_low_ram_pfn();
	else
		max_low_pfn = max_pfn;

	high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
#endif

    // 页表缓冲区申请
	early_alloc_pgt_buf();

	/*
	 * Need to conclude brk, before memblock_x86_fill()
	 *  it could use memblock_find_in_range, could overlap with
	 *  brk area.
	 */
	reserve_brk();

	cleanup_highmap();

	memblock_set_current_limit(ISA_END_ADDRESS);
	// memblock初始化
	memblock_x86_fill();
	
	// 建立低端内存和高端内存固定映射区的页表
	init_mem_mapping();
	// 内存管理框架初始化
	initmem_init();
	// 
	x86_init.paging.pagetable_init();
}
```

几乎所有的内存初始化工作都是在setup\_arch()中完成的，主要的工作包括：

（1）**建立内存图**：setup\_memory_map()；

（2）调用**e820\_end\_of\_ram\_pfn**()找出**最大可用页帧号max\_pfn**，32位情况下调用**find\_low\_pfn\_range**()找出**低端内存区**的**最大可用页帧号max\_low\_pfn**。

（3）初始化**memblock内存分配器**：memblock\_x86\_fill()；

（4）**初始化低端内存和高端内存固定映射区的页表**：init\_mem\_mapping()；

（5）**内存管理node节点设置**: initmem\_init()

（6）管理区和页面管理的构建: x86\_init.paging.pagetable\_init()

## 7.1 建立内存图

内存探测完之后，就要建立描述各**内存块情况**的**全局内存图结构**了。

```
start_kernel()
|
└->setup_arch()
   |
   └->setup_memory_map();   //arch/x86/kernel/e820.c
```

如下：

```c
void __init setup_memory_map(void)
{
	char *who;
	/* 调用x86体系下的memory_setup函数 */
	who = x86_init.resources.memory_setup();
	/* 保存到e820_saved中 */
	memcpy(&e820_saved, &e820, sizeof(struct e820map));
	printk(KERN_INFO "BIOS-provided physical RAM map:\n");
	/* 打印输出 */
	e820_print_map(who);
}
```

该函数调用x86\_init.resources.memory\_setup()实现对BIOS e820内存图的设置和优化，然后将**全局e820**中的值保存在**e820\_saved**中，并打印内存图。

Linux的**内存图**保存在一个**全局的e820变量**中，还有**其备份全局变量e820\_saved**，这**两个全局的e820map结构变量**均定义在**arch/x86/kernel/e820.c**中。**memory\_setup**()函数是**建立e820内存图**的**核心函数**，x86\_init.resources.**memory\_setup**()就是e820.c中的**default\_machine\_specific\_memory\_setup**()函数.

内存图设置函数**memory\_setup**()把**从BIOS中探测到的内存块**情况（保存在**boot\_params.e820\_map**中）做重叠检测，把**重叠的内存块去除**，然后调用**append\_e820\_map**()将它们添加到**全局的e820变量**中，具体完成添加工作的函数是\_\_**e820\_add_region**()。到这里，**物理内存**就已经从**BIOS中读出来**存放到**全局变量e820**中，e820是linux内核中用于建立内存管理框架的基础。例如建立初始化页表映射、管理区等都会用到它。

具体过程: 

将**boot\_params.e820\_map**各项的**起始地址**和**结束地址**和**change\_point**关联起来, 然后通过**sort进行排序**. **排序的结果**就是将**各项内存布局信息**所标示的**内存空间起始地址**和**结束地址**由低往高进行排序。如果**两者地址值相等**，则以两者的e820\_map项信息所标示的**内存空间尾做排序依据**，哪个空间尾**更后**，则该项排在**等值项后面**。

每个e820entry由两个change\_member表述

```
e820entry                         change_member(变量名字是change_point)
+-------------------+             +------------------------+
|addr, size         |             |start, e820entry        |
|                   |     ===>    +------------------------+
|type               |             |end,   e820entry        |
+-------------------+             +------------------------+
```

整个表形成了一个change\_member的数组

```
change_member*[]                               change_member[]
+------------------------+                     +------------------------+
|*start1                 |      ------>        |start1, entry1          |
+------------------------+                     +------------------------+
|*end1                   |      ------>        |end1,   entry1          |
+------------------------+                     +------------------------+
|*start2                 |      ------>        |start2, entry2          |
+------------------------+                     +------------------------+
|*end2                   |      ------>        |end2,   entry2          |
+------------------------+                     +------------------------+
|*start3                 |      ------>        |start3, entry3          |
+------------------------+                     +------------------------+
|*end3                   |      ------>        |end3,   entry3          |
+------------------------+                     +------------------------+
|*start4                 |      ------>        |start4, entry4          |
+------------------------+                     +------------------------+
|*end4                   |      ------>        |end4,   entry4          |
+------------------------+                     +------------------------+
```

对change\_member\-\>addr排序, 下面是排序后的change\_member\* 数组

```
change_member*[]                
+------------------------+      
|*start1                 |      
+------------------------+      
|*start2                 |      
+------------------------+      
|*end2                   |      
+------------------------+      
|*end1                   |      
+------------------------+      
|*start3                 |      
+------------------------+      
|*start4                 |      
+------------------------+      
|*end4                   |      
+------------------------+      
|*end3                   |      
+------------------------+   
```

**把已经排序完了的change\_point**做**整合**，将**重叠的内存空间根据属性进行筛选**，并将**同属性的相邻内存空间进行合并处理**。

![config](./images/15.png)

连续的同类型的合并到一块里面，不同类型的各自为政，不同类型重叠部分根据类型优先级高低拆分，依高优先级顺序保证各类型的内存块的完整性。

接下来就是将整理后的**boot\_params.e820\_map**添加到**全局变量数据e820**里

![config](./images/16.png)

## 7.2 页表缓冲区申请

在**setup\_arch**()函数中调用的**页表缓冲区申请**操作**early\_alloc\_pgt\_buf**()

注意该函数执行是在**memory\_block那些之前**

```
# /arch/x86/mm/init.c
void __init early_alloc_pgt_buf(void)
{
    unsigned long tables = INIT_PGT_BUF_SIZE;
    phys_addr_t base;
 
    base = __pa(extend_brk(tables, PAGE_SIZE));
 
    pgt_buf_start = base >> PAGE_SHIFT;
    pgt_buf_end = pgt_buf_start;
    pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);
}
```

从系统**开启分页管理（head\_32.s代码**）中**使用**到的\_\_**brk\_base保留空间申请一块内存**出来，申请的空间大小为：

```
#define INIT_PGT_BUF_SIZE        (6 * PAGE_SIZE)
```

也就是**24Kbyte**，同时将\_brk\_end标识的位置后移。里面涉及的几个全局变量作用：

- pgt\_buf\_start：标识**该缓冲空间的起始页框号**；

- pgt\_buf\_end：当前和pgt\_buf\_start等值，但是它用于表示该空间**未被申请使用的空间起始页框号**；

- pgt\_buf\_top：则是用来表示**缓冲空间的末尾**，存放的是该末尾的**页框号**。

在setup\_arch()中，紧接着early\_alloc\_pgt\_buf()还有reserve\_brk()：

```
# /arch/x86/kernel/setup.c

static void __init reserve_brk(void)
{
    if (_brk_end > _brk_start)
        memblock_reserve(__pa_symbol(_brk_start),
                 _brk_end - _brk_start);
 
    /* Mark brk area as locked down and no longer taking any
       new allocations */
    _brk_start = 0;
}
```

其主要是用来将**early\_alloc\_pgt\_buf**()申请的空间在memblock算法中做**reserved保留操作**，避免被其他地方申请使用引发异常。

## 7.3 memblock算法

memblock算法的实现是，**它将所有状态都保存在一个全局变量\_\_initdata\_memblock中，算法的初始化以及内存的申请释放都是在将内存块的状态做变更**。

### 7.3.1 struct memblock结构

那么从数据结构入手，\_\_initdata\_memblock是一个memblock结构体。其结构体定义：

```
# /include/linux/memblock.h
struct memblock {
    bool bottom_up;  /* is bottom up direction? 
    如果true, 则允许由下而上地分配内存*/
    phys_addr_t current_limit; /*指出了内存块的大小限制*/	
    /*  接下来的三个域描述了内存块的类型，即内存型, 预留型和物理内存*/
    struct memblock_type memory;
    struct memblock_type reserved;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
    struct memblock_type physmem;
#endif
};
```

该结构体包含五个域。

| 字段 | 描述 |
|:---:|:----|
| bottom\_up | 表示分配器分配内存的方式<br>true:从**低地址(内核映像的尾部**)向高地址分配<br>false:也就是top-down,从高地址向地址分配内存. |
| current\_limit | 指出了内存块的大小限制, 用于**限制通过memblock\_alloc等函数的内存申请** |
| memory | **可用内存的集合（不是说未分配的！！！而是所有的！！！**） |
| reserved | **已分配内存的集合** |
| physmem | **物理内存的集合**(需要配置CONFIG\_HAVE\_MEMBLOCK\_PHYS\_MAP参数) |


### 7.3.2 内存块类型struct memblock\_type

往下看看memory和reserved的结构体**memblock\_type**定义：

```
# /include/linux/memblock.h

struct memblock_type {
    unsigned long cnt; /* number of regions */
    unsigned long max; /* size of the allocated array */
    phys_addr_t total_size; /* size of all regions */
    struct memblock_region *regions;
};
```

该结构体存储的是**内存类型信息**

| 字段 | 描述 |
|:---:|:----:|
| cnt | 当前集合(memory或者reserved)中记录的**内存区域个数** |
| max | 当前集合(memory或者reserved)中可记录的**内存区域的最大个数** |
| total\_size | 集合记录**区域信息大小（region的size和，不是个数**） |
| regions | 内存区域结构指针 |

### 7.3.3 内存区域struct memblock\_region

```
# /include/linux/memblock.h

struct memblock_region {
    phys_addr_t base;
    phys_addr_t size;
    unsigned long flags;
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
    int nid;
#endif
};
```

| 字段 | 描述 |
|:---:|:----:|
| base | 内存区域起始地址 |
| size | 内存区域大小 |
| flags | 标记 |
| nid | **node号** |

### 7.3.4 内存区域标识

```cpp
// include/linux/memblock.h
/* Definition of memblock flags. */
enum {
    MEMBLOCK_NONE       = 0x0,  /* No special request */
    MEMBLOCK_HOTPLUG    = 0x1,  /* hotpluggable region */
    MEMBLOCK_MIRROR     = 0x2,  /* mirrored region */
    MEMBLOCK_NOMAP      = 0x4,  /* don't add to kernel direct mapping */
};
```

### 7.3.5 结构总体布局

结构关系图:

![config](./images/17.png)

一个memblock有3个memblock\_type，每个memblock\_type有一个memblock\_region指针

Memblock主要包含三个结构体：memblock,memblock\_type和memblock\_region。现在我们已了解了Memblock, 接下来我们将看到Memblock的初始化过程。

### 7.3.6 memblock初始化

#### 7.3.6.1 初始化memblock静态变量

在**编译（编译时确定！！！**）时,会分配好**memblock结构所需要的内存空间**. 

结构体memblock的初始化变量名和结构体名相同memblock：

```
# /mm/memblock.c

static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
static struct memblock_region memblock_physmem_init_regions[INIT_PHYSMEM_REGIONS] __initdata_memblock;
#endif

struct memblock memblock __initdata_memblock = {
    .memory.regions = memblock_memory_init_regions,
    .memory.cnt = 1, /* empty dummy entry */
    .memory.max = INIT_MEMBLOCK_REGIONS,
 
    .reserved.regions = memblock_reserved_init_regions,
    .reserved.cnt = 1, /* empty dummy entry */
    .reserved.max = INIT_MEMBLOCK_REGIONS,

#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
    .physmem.regions    = memblock_physmem_init_regions,
    .physmem.cnt        = 1,    /* empty dummy entry */
    .physmem.max        = INIT_PHYSMEM_REGIONS,
#endif

    .bottom_up = false,
    .current_limit = MEMBLOCK_ALLOC_ANYWHERE,
};
```

它初始化了部分成员，表示内存申请自高地址向低地址，且current_limit设为\~0，即0xFFFFFFFF，同时通过**全局变量定义为memblock**的算法管理中的memory和reserved**准备了内存空间**。

#### 7.3.6.2 宏\_\_initdata\_memblock指定了存储位置

```cpp
[include/linux/memblock.h]
#ifdef CONFIG_ARCH_DISCARD_MEMBLOCK
#define __init_memblock __meminit
#define __initdata_memblock __meminitdata
#else
#define __init_memblock
#define __initdata_memblock
#endif
```

启用CONFIG\_ARCH\_DISCARD\_MEMBLOCK宏配置选项，memblock代码会被放**到\.init代码段**,在内核启动完成后memblock代码会从.init代码段释放。

memblock结构体中3个memblock\_type类型数据 **memory**, **reserved**和**physmem**的初始化

它们的**memblock\_type cnt域**(当前集合中区域个数)被初始化为**1**。**memblock\_type max域**(当前集合中**最大区域个数**)被初始化为`INIT_MEMBLOCK_REGIONS`和`INIT_PHYSMEM_REGIONS`

其中`INIT_MEMBLOCK_REGIONS`为128, 参见[include/linux/memblock.h?v=4.7, line 20](http://lxr.free-electrons.com/source/include/linux/memblock.h?v=4.7#L20)

```cpp
#define INIT_MEMBLOCK_REGIONS   128
#define INIT_PHYSMEM_REGIONS    4
```

而**memblock\_type.regions**域都是通过memblock\_region数组初始化的,所有的数组定义都带有\_\_initdata\_memblock宏

memblock结构体中最后两个域**bottom\_up**,**内存分配模式(从低地址往高地址！！！）被禁用**(bottom\_up = **false**, 因此内存分配方式为top-down.), 当前memblock的大小限制(memblock\_alloc等的）是`MEMBLOCK\_ALLOC_ANYWHERE`为\~(phys\_addr\_t)0即为**0xffffffff(32个1**).

```cpp
/* Flags for memblock_alloc_base() amd __memblock_alloc_base() */
#define MEMBLOCK_ALLOC_ANYWHERE (~(phys_addr_t)0)
#define MEMBLOCK_ALLOC_ACCESSIBLE       0
```

#### 7.3.6.3 x86架构下的memblock初始化

在物理内存探测并且整理保存到全局变量e820后, memblock初始化发生在这个之后

```cpp
void __init setup_arch(char **cmdline_p)
{
    /*
     * Need to conclude brk, before memblock_x86_fill()
     *  it could use memblock_find_in_range, could overlap with
     *  brk area.
     */
    reserve_brk();

    cleanup_highmap();

    memblock_set_current_limit(ISA_END_ADDRESS);
    memblock_x86_fill();
}
```

```c
static void __init reserve_brk(void)
{
    if (_brk_end > _brk_start)
        memblock_reserve(__pa_symbol(_brk_start),
                 _brk_end - _brk_start);

    /* Mark brk area as locked down and no longer taking any
       new allocations */
    _brk_start = 0;
}
```

首先**内核建立内核页表**需要**扩展\_\_brk**,而**扩展后的brk**就立即**被声明为已分配**. 这项工作是由**reserve\_brk**通过调用**memblock\_reserve(这也就是第一阶段的使用**)完成的，而其实**并不是真正通过memblock分配**的,因为此时**memblock还没有完成初始化**

此时memblock还没有初始化,只能通过memblock\_reserve来完成内存的分配

函数实现：

```
# /arch/x86/kernel/e820.c

void __init memblock_x86_fill(void)
{
    int i;
    u64 end;
    memblock_allow_resize();
    for (i = 0; i < e820.nr_map; i++) {
        struct e820entry *ei = &e820.map[i];
        end = ei->addr + ei->size;
        if (end != (resource_size_t)end)
            continue;
        if (ei->type != E820_RAM && ei->type != E820_RESERVED_KERN)
            continue;
        memblock_add(ei->addr, ei->size);
    }
    /* throw away partial pages */
    memblock_trim_memory(PAGE_SIZE);
    memblock_dump_all();
}
```

遍历之前的**全局变量e820**的内存布局信息, 只对**E820\_RAM**和**E820\_RESERVED\_KERN**类型的进行添加到系统变量memblock中的**memory**中（**这儿参见上面memblock\_add，其实现中使用nid是2\^CONFIG\_NODES\_SHIFT！！！初始化这儿是不区分node的！！！**）, 当做可分配内存, 添加时候如果memory不为空, 需要检查内存重叠情况并剔除, 然后通过**memblock\_merge\_regions**()把紧挨着的内存合并.

**后两个函数**主要是修剪内存**使之对齐和输出信息**.

### 7.3.7 memblock操作总结

memblock内存管理是将**所有的物理内存**放到**memblock.memory**中作为可用内存来管理, **分配过**的内存**只加入**到**memblock.reserved**中,并**不从memory中移出**, 最后都通过memblock\_merge\_regions()把紧挨着的内存合并了.

同理**释放内存**仅仅从**reserved**中移除.也就是说,**memory**在**fill过后**基本就是**不动**的了. **申请和分配内存**仅仅修改**reserved**就达到目的.

**memory链表**维护系统的**内存信息**(在初始化阶段**通过bios获取**的),对于任何**内存分配**,先去**查找memory链表**,然后**在reserve链表上记录**(新增一个节点，或者合并)

- 可以分配**小于一页的内存**; 

- 从高往低找, **就近查找可用的内存**

## 7.4 建立内核页表

**32位**情况下, **每个进程**一般都能寻址**4G的内存空间**. 但如果**物理内存没这么大**的话, 进程怎么能获得4G的内存空间呢？这就是使用了**虚拟地址**的好处。我们经常在程序的反汇编代码中看到一些类似0x32118965这样的地址，**操作系统**中称为**线性地址**，或**虚拟地址**。通常我们使用一种叫做**虚拟内存的技术**来实现，因为可以**使用硬盘中的一部分来当作内存**使用。另外，现在操作系统都划分为**系统空间**和**用户空间**，使用**虚拟地址**可以很好的**保护内核空间不被用户空间破坏**。Linux 2.6内核使用了许多技术来改进对大量虚拟内存空间的使用，以及对内存映射的优化，使得Linux比以往任何时候都更适用于企业。包括**反向映射（reverse mapping**）、使用**更大的内存页**、**页表条目存储在高端内存**中，以及更稳定的管理器。对于**虚拟地址**如何**转为物理地址**,这个**转换过程**有**操作系统**和**CPU**共同完成。**操作系统**为CPU**设置好页表**。**CPU**通过**MMU单元**进行**地址转换**。**CPU**做出**映射**的**前提**是**操作系统**要为其**准备好内核页表**，而对于**页表的设置**，内核在**系统启动的初期(！！！**)和**系统初始化完成后(！！！**)都分别进行了设置。

Linux简化了分段机制，使得虚拟地址与线性地址总是一致，因此Linux的虚拟地址空间也为0～4G. Linux内核将这4G字节的空间分为两部分。将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF）供内核使用，称为“内核空间”。而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF）供各个进程使用，称为“用户空间“。因为每个进程可以通过系统调用进入内核，因此**Linux内核**由**系统内的所有进程共享**。于是，从具体进程的角度来看，每个进程可以拥有4G字节的虚拟空间。

Linux使用两级保护机制：**0级供内核**使用，**3级供用户程序**使用。**每个进程**有各自的**私有用户空间（0～3G**），这个空间对系统中的**其他进程是不可见**的。**最高的1GB字节虚拟内核空间**则为**所有进程以及内核所共享**。**内核空间(！！！**)中存放的是**内核代码和数据(！！！**)，而**进程的用户空间**中存放的是**用户程序的代码和数据**。不管是内核空间还是用户空间，它们都处于虚拟空间中。虽然**内核空间**占据了**每个虚拟空间**中的**最高1GB字节(！！！**)，但映射到**物理内存**却总是**从最低地址（0x00000000）开始**。对**内核空间**来说，其**地址映射是很简单的线性映射**，**0xC0000000**就是**物理地址与线性地址之间的位移量**，在Linux代码中就叫做**PAGE\_OFFSET**。

Linux启动并建立一套**完整的页表机制**要经过以下几个步骤：

1. **临时内核页表的初始化**(setup\_32.s)

2. 启动**分页机制(head\_32.s**)

3. 建立**低端内存**和**高端内存固定映射区**的**页表**(**init\_memory\_mapping**())

4. 建立**高端内存永久映射区**的**页表**并获取**固定映射区的临时映射区页表**(paging\_init())

linux**页表映射机制**的建立分为**两个阶段**，

第一个阶段是**内核进入保护模式之前**要先建立一个**临时内核页表**并**开启分页**功能，因为在**进入保护模式后**，内核**继续初始化**直到建立**完整的内存映射机制之前**，仍然需要用到**页表**来映射相应的内存地址。对**x86 32**位内核，这个工作在保护模式下的内核入口函数arch/x86/kernel/head\_32.S:startup\_32()中完成。

第二阶段是**建立完整的内存映射机制**，在在setup\_arch()--->arch/x86/mm/init.c:**init\_memory\_mapping**()中完成。注意对于**物理地址扩展（PAE)分页机制**，Intel通过在她得处理器上把管脚数从32增加到36已经满足了这些需求，**寻址能力**可以达到**64GB**。不过，只有引入一种新的分页机制把32位线性地址转换为36位物理地址才能使用所增加的物理地址。linux为对多种体系的支持，选择了一套简单的通用实现机制。在这里只分析x86 32位下的实现。

### 7.4.1 临时页表的初始化

**swapper\_pg\_dir**是**临时全局页目录表起址**，它是在**内核编译过程**中**静态初始化**的。内核是在swapper\_pg\_dir的第**768个表项**开始建立页表。其**对应线性地址**就是\_\_**brk\_base**（内核编译时指定其值，默认为**0xc0000000**）以上的地址，即**3GB以上的高端地址**（3GB\-4GB），再次强调这高端的1GB线性空间是内核占据的虚拟空间，在进行实际内存映射时，映射到**物理内存**却总是从最低地址（**0x00000000**）开始。

内核从\_\_**brk\_base开始建立页表**, 然后创建页表相关结构, **开启CPU映射机制**, 继续初始化(包括INIT\_TASK<即第一个进程>, 建立完整中断处理程序, 中心加载GDT描述符), 最后**跳转到init/main.c中的start\_kernel()函数继续初始化**.

### 7.4.2 内存映射机制的完整建立初始化

这一阶段在start\_kernel()--->**setup\_arch**()中完成. 在Linux中，**物理内存**被分为**低端内存区**和**高端内存区**（如果内核编译时**配置了高端内存标志**的话），为了**建立物理内存到虚拟地址空间的映射**，需要先计算出**物理内存总共有多少页面数**，即找出**最大可用页框号**，这**包含了整个低端和高端内存区**。还要计算出**低端内存区总共占多少页面**。

下面就基于RAM大于896MB，而小于4GB，并且**CONFIG\_HIGHMEM(必须配置！！！**）配置了高端内存的环境情况进行分析。

#### 7.4.2.1 相关变量与宏的定义

- max\_pfn：**最大物理内存页面帧号**；

- max\_low\_pfn：**低端内存区（直接映射空间区的内存）的最大可用页帧号**；

在**setup\_arch**()，首先调用arch/x86/kernel/e820.c:**e820\_end\_of\_ram\_pfn**()找出**最大可用页帧号（即总页面数**），并保存在**全局变量max\_pfn**中，这个变量定义可以在mm/bootmem.c中找到。它直接调用e820.c中的e820\_end\_pfn()完成工作。

**e820\_end\_of\_ram\_pfn**()直接调用e820\_end\_pfn()找出**最大可用页面帧号**，它会**遍历e820.map数组**中存放的**所有物理页面块**，找出其中**最大的页面帧号**，这就是我们当前需要的**max\_pfn值**。

setup\_arch()会调用arch/x86/mm/init\_32.c:**find\_low\_pfn\_range**()找出**低端内存区**的**最大可用页帧号**，保存在**全局变量max\_low\_pfn**中（也定义在mm/bootmem.c中）。

```
start_kernel()                      #/init/main.c
|
└─>setup_arch()                     #/arch/x86/kernel/setup.c
   |
   └─>e820_end_of_ram_pfn()         #/arch/x86/kernel/e820.c
   |
   └─>find_low_pfn_range()          #/arch/x86/kernel/e820.c
```

```c
/*
 * Determine low and high memory ranges:
 */
void __init find_low_pfn_range(void)
{
    /* it could update max_pfn */
 
    if (max_pfn <= MAXMEM_PFN)
        lowmem_pfn_init();
    else
        highmem_pfn_init();
}
```

根据**max\_pfn**是否大于**MAXMEM\_PFN**，从而判断**是否初始化高端内存**

Linux支持4级页表, 根据上面讲过的PAGE\_SIZE等会逐步推出诸如FIXADDR\_BOOT\_START, PKMAP\_BASE等数值

![config](./images/18.png)

![config](./images/19.png)

#### 7.4.2.3 低端内存页表和高端内存固定映射区页表的建立init\_mem\_mapping()

有了**总页面数**、**低端页面数**、**高端页面数**这些信息，setup\_arch()接着调用arch/x86/mm/init.c:**init\_mem\_mapping**()函数**建立低端内存页表和高端内存固定映射区的页表**.

该函数**在PAGE\_OFFSET处(！！！**)建立**物理内存的直接映射**，即**把物理内存中0\~max\_low\_pfn\<\<12**地址范围的**低端空间区直接映射**到**内核虚拟空间**（它是从**PAGE\_OFFSET**即**0xc0000000**开始的**1GB线性地址**）。这在**bootmem/memblock初始化之前运行**，并且**直接从物理内存获取页面**，这些页面在前面已经被**临时映射**了。注意高端映射区并没有映射到实际的物理页面，只是这种机制的初步建立，页表存储的空间保留。

调用关系如下：

```cpp
setup_arch()
|
|-->init_mem_mapping()  //低端内存页表和高端内存固定映射区的页表
    |
    |-->probe_page_size_mask() //
    |
    |-->init_memory_mapping(0, ISA_END_ADDRESS);
    |
    |-->early_ioremap_page_table_range_init()  // 高端内存的固定映射区
    |
    |-->load_cr3(swapper_pg_dir);  //将内核PGD地址加载到cr3寄存器
```

## 7.5 内存管理node节点设置initmem\_init()

```c
[arch/x86/mm/init_64.c]
#ifndef CONFIG_NUMA
void __init initmem_init(void)
{
	memblock_set_node(0, (phys_addr_t)ULLONG_MAX, &memblock.memory, 0);
}
#endif

[arch/x86/mm/numa_64.c]
void __init initmem_init(void)
{
	x86_numa_init();
}
```

上面是针对非NUMA情况, 下面是numa的初始化

memblock\_set\_node, 该函数用于给早前建立的memblock算法**设置node节点信息**。这里传参数是**全的memblock.memory信息**.

linux内核中是如何获得NUMA信息的

在x86平台，这个工作分成两步

- 将numa信息保存到numa\_meminfo
- 将numa\_meminfo映射到memblock结构

着重关注第一次获取到numa信息的过程，对node和zone的数据结构暂时不在本文中体现。

整体调用结构

```c
setup_arch()
  initmem_init()
    x86_numa_init()
        numa_init()
            x86_acpi_numa_init()
            numa_cleanup_meminfo()
            numa_register_memblks()
                memblock_set_node()
                alloc_node_data()
                memblock_dump_all()
```

### 7.5.1 将numa信息保存到numa\_meminfo

在x86架构上，**numa信息第一次获取**是通过**acpi**或者是**读取北桥上的信息**。具体的函数是**numa\_init**()。不管是哪种方式，**numa相关的信息**都最后保存在了**numa\_meminfo这个数据结构**中。

这个数据结构和memblock长得很像，展开看就是一个数组，**每个元素**记录了**一段内存的起止地址和node信息**。

```
numa_meminfo
    +------------------------------+
    |nr_blks                       |
    |    (int)                     |
    +------------------------------+
    |blk[NR_NODE_MEMBLKS]          |
    |    (struct numa_memblk)      |
    |    +-------------------------+
    |    |start                    |
    |    |end                      |
    |    |   (u64)                 |
    |    |nid                      |
    |    |   (int)                 |
    +----+-------------------------+
```

在这个过程中使用的就是numa\_add\_memblk()函数添加的numa\_meminfo数据结构。

### 7.5.2 将numa\_meminfo映射到memblock结构

内核获取了**numa\_meminfo**之后并没有如我们想象一般直接拿来用了。虽然此时**给每个numa节点**生成了我们以后会看到的**node\_data数据结构**，但此时并没有直接使能它。

memblock是内核初期内存分配器，所以当内核获取了**numa信息**首先是**把相关的信息映射到了memblock结构**中，使其具有numa的knowledge。这样在**内核初期分配内存**时，也可以分配到更近的内存了。

在这个过程中有两个比较重要的函数

- numa\_cleanup\_meminfo()
- numa\_register\_memblks()

前者主要用来**过滤numa\_meminfo结构**，**合并同一个node上的内存**。

后者就是**把numa信息映射到memblock**了。除此之外，顺便还把之后需要的**node\_data给分配(alloc\_node\_data**)了，为后续的页分配器做好了准备。

```c
static int __init numa_register_memblks(struct numa_meminfo *mi)
{
    for (i = 0; i < mi->nr_blks; i++) {
		struct numa_memblk *mb = &mi->blk[i];
		memblock_set_node(mb->start, mb->end - mb->start,
				  &memblock.memory, mb->nid);
	}
	
	/* Finally register nodes. */
	for_each_node_mask(nid, node_possible_map) {
		u64 start = PFN_PHYS(max_pfn);
		u64 end = 0;

		for (i = 0; i < mi->nr_blks; i++) {
			if (nid != mi->blk[i].nid)
				continue;
			start = min(mi->blk[i].start, start);
			end = max(mi->blk[i].end, end);
		}

		if (start >= end)
			continue;

		/*
		 * Don't confuse VM with a node that doesn't have the
		 * minimum amount of memory:
		 */
		if (end && (end - start) < NODE_MIN_SIZE)
			continue;

		alloc_node_data(nid);
	}	
	/* Dump memblock with node info and return. */
	memblock_dump_all();
	return 0;
}
```

memblock\_set\_node主要调用了三个函数做相关操作：memblock\_isolate\_range、memblock\_set\_region\_node和memblock\_merge\_regions。

```c
【file：/mm/memblock.c】
/**
 * memblock_set_node - set node ID on memblock regions
 * @base: base of area to set node ID for
 * @size: size of area to set node ID for
 * @type: memblock type to set node ID for
 * @nid: node ID to set
 *
 * Set the nid of memblock @type regions in [@base,@base+@size) to @nid.
 * Regions which cross the area boundaries are split as necessary.
 *
 * RETURNS:
 * 0 on success, -errno on failure.
 */
int __init_memblock memblock_set_node(phys_addr_t base, phys_addr_t size,
                      struct memblock_type *type, int nid)
{
    int start_rgn, end_rgn;
    int i, ret;
 
    ret = memblock_isolate_range(type, base, size, &start_rgn, &end_rgn);
    if (ret)
        return ret;
 
    for (i = start_rgn; i < end_rgn; i++)
        memblock_set_region_node(&type->regions[i], nid);
 
    memblock_merge_regions(type);
    return 0;
}
```

**memblock\_isolate\_range**主要做**分割操作**, 在**memblock算法建立时**，只是**判断了flags是否相同**，然后将**连续内存做合并**操作，而**此时建立node节点**，则根据**入参base和size标记节点内存范围**将**内存划分开来**。

如果**memblock中的region**恰好以**该节点内存范围末尾划分开来**的话，那么则将region的索引记录至start\_rgn，索引加1记录至end\_rgn返回回去；

如果**memblock中的region**跨越了**该节点内存末尾分界**，那么将会把**当前的region边界调整为node节点内存范围边界**，另一部分通过**memblock\_insert\_region**()函数**插入到memblock管理regions当中**，以完成拆分。

memblock\_set\_region\_node是获取node节点号，而memblock\_merge\_regions()则是用于将region合并的。

### 7.5.3 观察memblock的变化

memblock的调试信息默认没有打开，所以要观察的话需要传入内核启动参数”memblock=debug”。

进入系统后，输入命令”dmesg | grep -A 9 MEMBLOCK”可以看到

## 7.6 管理区和页面管理的构建x86\_init.paging.pagetable\_init()

x86\_init.paging.pagetable\_init(), 该钩子实际上挂接的是native\_pagetable\_init()函数。

[arch/x86/mm/init\_32.c]

(1) 循环检测**max\_low\_pfn直接映射空间后面**的**物理内存**是否存在**系统启动引导时创建的页表**，如果存在，则使用pte\_clear()将其清除。

(2) 接下来的paravirt\_alloc\_pmd()主要是用于准虚拟化，主要是使用钩子函数的方式替换x86环境中多种多样的指令实现。

(3) 再往下的paging\_init()

### 7.6.1 paging\_init()

```c
[arch/x86/mm/init_64.c]
void __init paging_init(void)
{
	sparse_memory_present_with_active_regions(MAX_NUMNODES);
	sparse_init();

	/*
	 * clear the default setting with node 0
	 * note: don't use nodes_clear here, that is really clearing when
	 *	 numa support is not compiled in, and later node_set_state
	 *	 will not set it back.
	 */
	node_clear_state(0, N_MEMORY);
	if (N_MEMORY != N_NORMAL_MEMORY)
		node_clear_state(0, N_NORMAL_MEMORY);

	zone_sizes_init();
}
```

这里**sparse memory**涉及到linux的一个**内存模型概念**。linux内核有**三种内存模型**：**Flat memory**、**Discontiguous memory**和**Sparse memory**。其分别表示：

**Flat memory**：顾名思义，**物理内存是平坦连续的**，整个系统只有**一个node**节点。

**Discontiguous memory**：**物理内存不连续**，内存中**存在空洞**，也因而系统**将物理内存分为多个节点**，但是**每个节点的内部内存是平坦连续**的。值得注意的是，该模型不仅是对于*NUMA环境*而言，**UMA**环境上同样可能存在**多个节点**的情况。

**Sparse memory**：**物理内存是不连续**的，**节点的内部内存也可能是不连续**的，系统也因而可能会有**一个或多个节点**。此外，该模型是**内存热插拔**的基础。


........


**zone\_size\_init**

```c
void __init zone_sizes_init(void)
{
	unsigned long max_zone_pfns[MAX_NR_ZONES];

	memset(max_zone_pfns, 0, sizeof(max_zone_pfns));

#ifdef CONFIG_ZONE_DMA
	max_zone_pfns[ZONE_DMA]		= min(MAX_DMA_PFN, max_low_pfn);
#endif
#ifdef CONFIG_ZONE_DMA32
	max_zone_pfns[ZONE_DMA32]	= min(MAX_DMA32_PFN, max_low_pfn);
#endif
	max_zone_pfns[ZONE_NORMAL]	= max_low_pfn;
#ifdef CONFIG_HIGHMEM
	max_zone_pfns[ZONE_HIGHMEM]	= max_pfn;
#endif

	free_area_init_nodes(max_zone_pfns);
}
```

通过max\_zone\_pfns获取各个管理区的最大页面数，并作为参数调用free\_area\_init\_nodes()

#### 7.6.1.1 free\_area\_init\_nodes()

[/mm/page_alloc.c]

该函数中，**arch\_zone\_lowest\_possible\_pfn**用于存储**各内存管理区**可使用的**最小内存页框号**，而**arch\_zone\_highest\_possible\_pfn**则是用来存储**各内存管理区**可使用的**最大内存页框号**。也就是说**确定了各管理区的上下边界**. 此外，还有一个**全局数组zone\_movable\_pfn**，用于记录**各个node**节点的**Movable管理区的起始页框号**

打印管理区范围信息(dmesg可看到)

setup\_nr\_node\_ids()设置内存节点总数

最后有一个**遍历各个节点**做初始化

```c
for_each_online_node(nid) {
    pg_data_t *pgdat = NODE_DATA(nid);
    free_area_init_node(nid, NULL,
            find_min_pfn_for_node(nid), NULL);

    /* Any memory on that node */
    if (pgdat->node_present_pages)
        node_set_state(nid, N_MEMORY);
    check_for_memory(pgdat, nid);
}
```

node\_set\_state()主要是对node节点进行状态设置，而check\_for\_memory()则是做内存检查。

关键函数是**free\_area\_init\_node**()，其入参find\_min\_pfn\_for\_node()用于**获取node**节点中**最低的内存页框号**。

```c
【file：/mm/page_alloc.c】
void __paginginit free_area_init_node(int nid, unsigned long *zones_size,
        unsigned long node_start_pfn, unsigned long *zholes_size)
{
    pg_data_t *pgdat = NODE_DATA(nid);
    unsigned long start_pfn = 0;
    unsigned long end_pfn = 0;
 
    /* pg_data_t should be reset to zero when it's allocated */
    WARN_ON(pgdat->nr_zones || pgdat->classzone_idx);
 
    pgdat->node_id = nid;
    pgdat->node_start_pfn = node_start_pfn;
    init_zone_allows_reclaim(nid);
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
    get_pfn_range_for_nid(nid, &start_pfn, &end_pfn);
#endif
    calculate_node_totalpages(pgdat, start_pfn, end_pfn,
                  zones_size, zholes_size);
 
    alloc_node_mem_map(pgdat);
#ifdef CONFIG_FLAT_NODE_MEM_MAP
    printk(KERN_DEBUG "free_area_init_node: node %d, pgdat %08lx, node_mem_map %08lx\n",
        nid, (unsigned long)pgdat,
        (unsigned long)pgdat->node_mem_map);
#endif
 
    free_area_init_core(pgdat, start_pfn, end_pfn,
                zones_size, zholes_size);
}
```

- init\_zone\_allows\_reclaim()评估内存管理区是否可回收以及合适的node节点数
- get\_pfn\_range\_for\_nid获取内存node节点的起始和末尾页框号
- calculate\_node\_totalpages(): 遍历node的zone, 得到所有zone的所有页面数(node\_spanned\_pages), 不包括movable管理区; 计算内存空洞页面数; 从而得到物理页面总数(node\_present\_pages); 打印节点信息和node\_present\_pages
- alloc\_node\_mem\_map(): 给**当前节点的内存页面**信息**申请内存空间**, 并赋值给pgdat\->node\_mem\_map; 如果当前节点是0号节点, 设置**全局变量mem\_map**为当前节点的node\_mem\_map
- free\_area\_init\_core: 初始化工作

设置了内存管理节点的管理结构体，包括pgdat\_resize\_init()初始化**锁**资源、init\_waitqueue\_head()初始**内存队列**、pgdat\_page\_cgroup\_init()**控制组群初始化**。

循环遍历统计**各个管理区**最大跨度间相差的**页面数size**以及**除去内存“空洞**”**后的实际页面数realsize**,然后通过**calc\_memmap\_size**()计算出**该管理区**所需的**页面管理结构**占用的**页面数memmap\_pages**，最后可以计算得除高端内存外的系统内存共有的**内存页面数nr\_kernel\_pages（用于统计所有一致映射的页**）；此外循环体内的操作则是**初始化内存管理区的管理结构(zone的初始化**)，例如各类锁的初始化、队列初始化。值得注意的是**zone\_pcp\_init**()是初始化**冷热页分配器**的，mod\_zone\_page\_state()用于**计算更新管理区的状态统计**，lruvec\_init()则是**初始化LRU算法使用的链表和保护锁**，而set\_pageblock\_order()用于在CONFIG\_HUGETLB\_PAGE\_SIZE\_VARIABLE配置下设置pageblock\_order值的；此外**setup\_usemap**()函数则是主要是为了给zone管理结构体中的**pageblock\_flags**申请**内存空间**，pageblock\_flags与**伙伴系统的碎片迁移算法有关**。而init\_currently\_empty\_zone()则主要初始化管理区的**等待队列哈希表**和**等待队列**，同时还初始化了**与伙伴系统相关的free\_aera列表**; memmap\_init: 根据页框号pfn通过pfn\_to\_page()查找到页面管理结构page, 然后对其进行初始化.

中间有部分记录可以通过demesg查到

至此, 内存管理框架构建完毕。

# 8 build\_all\_zonelists初始化每个node的备用管理区链表zonelists

注: 该备用列表必须包括**所有结点(！！！包括当前节点！！！**)的**所有内存域(！！！**)

为内存管理做得一个准备工作就是将所有节点的管理区（所有的节点pg_data_t的zone！！！）都链入到zonelist中，便于后面内存分配工作的进行.

内存节点pg\_data\_t中将内存节点中的内存区域zone按照某种组织层次（可配置！！！）存储在一个zonelist中, 即**pglist\_data\->node\_zonelists成员信息**

```c
//  http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L626
typedef struct pglist_data
{
	struct zone node_zones[MAX_NR_ZONES];
	struct zonelist node_zonelists[MAX_ZONELISTS];
}
```

内核定义了内存的一个层次结构关系, 首先试图分配廉价的内存，如果失败，则根据访问速度和容量，逐渐尝试分配更昂贵的内存.

高端内存最廉价, 因为内核没有任何部分依赖于从该内存域分配的内存,如果高端内存用尽,对内核没有副作用, 所以优先分配高端内存

普通内存域的情况有所不同, 许多内核数据结构必须保存在该内存域, 而不能放置到高端内存域, 因此如果普通内存域用尽, 那么内核会面临内存紧张的情况

DMA内存域最昂贵，因为它用于外设和系统之间的数据传输。

举例来讲，如果内核指定想要分配高端内存域。它首先在当前结点的高端内存域寻找适当的空闲内存段，如果失败，则查看该结点的普通内存域，如果还失败，则试图在该结点的DMA内存域分配。如果在3个本地内存域都无法找到空闲内存，则查看其他结点。这种情况下，备选结点应该尽可能靠近主结点，以最小化访问非本地内存引起的性能损失。

start\_kernel()接下来的初始化则是linux**通用的内存管理算法框架**了。

之前已经完成了节点和管理区的关键数据的初始化.

build\_all\_zonelists()用来初始化**内存分配器**使用的**存储节点**中的**管理区链表node\_zonelists**，是为**内存管理算法（伙伴管理算法**）做准备工作的。

```c
build_all_zonelists(NULL, NULL);
```

函数实现

```cpp
void __ref build_all_zonelists(pg_data_t *pgdat, struct zone *zone)
{
	/*  设置zonelist中节点和内存域的组织形式
     *  current_zonelist_order变量标识了当前系统的内存组织形式
     *	zonelist_order_name以字符串存储了系统中内存组织形式的名称  */
    set_zonelist_order();

    if (system_state == SYSTEM_BOOTING) {
        build_all_zonelists_init();
    } else {
#ifdef CONFIG_MEMORY_HOTPLUG
        if (zone)
            setup_zone_pageset(zone);
#endif
        stop_machine(__build_all_zonelists, pgdat, NULL);
    }
    vm_total_pages = nr_free_pagecache_pages();
    if (vm_total_pages < (pageblock_nr_pages * MIGRATE_TYPES))
        page_group_by_mobility_disabled = 1;
    else
        page_group_by_mobility_disabled = 0;

    pr_info("Built %i zonelists in %s order, mobility grouping %s.  Total pages: %ld\n",
        nr_online_nodes,
        zonelist_order_name[current_zonelist_order],
        page_group_by_mobility_disabled ? "off" : "on",
        vm_total_pages);
#ifdef CONFIG_NUMA
    pr_info("Policy zone: %s\n", zone_names[policy_zone]);
#endif
}
```

## 8.1 设置结点初始化顺序set\_zonelist\_order()

可以通过启动参数"**numa\_zonelist\_order**"来配置zonelist order，内核定义了3种配置

```cpp
// http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L4551
#define ZONELIST_ORDER_DEFAULT  0 /* 智能选择Node或Zone方式 */
#define ZONELIST_ORDER_NODE     1 /* 对应Node方式 */
#define ZONELIST_ORDER_ZONE     2 /* 对应Zone方式 */
```

非NUMA系统中, 这两个排序方式都是一样的, 也就是legacy方式.

**全局的current\_zonelist\_order变量**标识了系统中的**当前使用的内存域排列方式**, 默认配置为ZONELIST\_ORDER\_DEFAULT

```
static int current_zonelist_order = ZONELIST_ORDER_DEFAULT;
static char zonelist_order_name[3][8] = {"Default", "Node", "Zone"};
```

流程:

1. 非NUMA系统, current\_zonelist\_order为ZONE方式(与NODE方式相同)

2. NUMA系统, 根据设置

- 如果是ZONELIST\_ORDER\_DEFAULT, 则内核选择, 目前是32位系统ZONE方式, 64位系统NODE方式

可以通过/proc/sys/vm/numa\_zonelist\_order动态改变zonelist order的分配方式

## 8.2 system\_state系统状态标识

其中**system\_state**变量是一个**系统全局定义**的用来表示**系统当前运行状态**的枚举变量

```cpp
[include/linux/kernel.h]
extern enum system_states
{
	SYSTEM_BOOTING,
	SYSTEM_RUNNING,
	SYSTEM_HALT,
	SYSTEM_POWER_OFF,
	SYSTEM_RESTART,
} system_state;
```

系统状态system\_state为SYSTEM\_BOOTING，系统状态只有在**start\_kernel**执行到最后一个函数**rest\_init**后，才会进入**SYSTEM\_RUNNING**

## 8.3 build\_all\_zonelists\_init函数

```cpp
[mm/page_alloc.c]
static noinline void __init
build_all_zonelists_init(void)
{
    __build_all_zonelists(NULL);
    mminit_verify_zonelist();
    cpuset_init_current_mems_allowed();
}
```

build\_all\_zonelists\_init将将所有工作都委托给\_\_build\_all\_zonelists完成了zonelists的初始化工作, 传入的参数为NULL.

```cpp
static int __build_all_zonelists(void *data)
{
    int nid;
    int cpu;
    pg_data_t *self = data;

	/*  ......  */

    for_each_online_node(nid) {
        pg_data_t *pgdat = NODE_DATA(nid);

        build_zonelists(pgdat);
    }
    
    for_each_possible_cpu(cpu) {
		setup_pageset(&per_cpu(boot_pageset, cpu), 0);
	/*  ......  */
}
```

**遍历**了系统中**所有的活动结点(所有的, 不仅仅是其他节点**), **每次调用**对**一个不同结点**生成**内存域数据**

### 8.3.1 build\_zonelists初始化每个内存结点的zonelists

**遍历每个node**, build\_zonelists(pg\_data\_t \*pgdat)完成了**节点pgdat**上**zonelists的初始化**工作,它建立了**备用层次结构zonelists**. 

由于UMA和NUMA架构下结点的层次结构有很大的区别,因此内核分别提供了两套不同的接口.

```c
[mm/page_alloc.c]
#ifdef CONFIG_NUMA

static int __parse_numa_zonelist_order(char *s)

static __init int setup_numa_zonelist_order(char *s)

int numa_zonelist_order_handler(struct ctl_table *table, int write,
             void __user *buffer, size_t *length,

static int find_next_best_node(int node, nodemask_t *used_node_mask)

static void build_zonelists_in_node_order(pg_data_t *pgdat, int node)

static void build_thisnode_zonelists(pg_data_t *pgdat)

static void build_zonelists_in_zone_order(pg_data_t *pgdat, int nr_nodes)

#if defined(CONFIG_64BIT)

static int default_zonelist_order(void)

#else

static int default_zonelist_order(void)

#endif /* CONFIG_64BIT */

static void build_zonelists(pg_data_t *pgdat)

#ifdef CONFIG_HAVE_MEMORYLESS_NODES

int local_memory_node(int node)

#endif

#else   /* CONFIG_NUMA */

static void build_zonelists(pg_data_t *pgdat)

static void set_zonelist_order(void)

#endif  /* CONFIG_NUMA */
```

以**UMA结构**下的build\_zonelists为例, 来讲讲内核是怎么初始化**备用内存域层次结构**的

```c
static void build_zonelists(pg_data_t *pgdat)
{
	int node, local_node;
	enum zone_type j;
	struct zonelist *zonelist;

	local_node = pgdat->node_id;

	zonelist = &pgdat->node_zonelists[0];
	j = build_zonelists_node(pgdat, zonelist, 0);

	/*
	 * Now we build the zonelist so that it contains the zones
	 * of all the other nodes.
	 * We don't want to pressure a particular node, so when
	 * building the zones for node N, we make sure that the
	 * zones coming right after the local ones are those from
	 * node N+1 (modulo N)
	 */
	for (node = local_node + 1; node < MAX_NUMNODES; node++) {
		if (!node_online(node))
			continue;
		j = build_zonelists_node(NODE_DATA(node), zonelist, j);
	}
	for (node = 0; node < local_node; node++) {
		if (!node_online(node))
			continue;
		j = build_zonelists_node(NODE_DATA(node), zonelist, j);
	}

	zonelist->_zonerefs[j].zone = NULL;
	zonelist->_zonerefs[j].zone_idx = 0;
}
```

node\_zonelists的数组元素通过指针操作寻址. 实际工作则委托给build\_zonelist\_node。

UMA结构下, 数组大小MAX\_ZONELISTS = 1, 所以使用**zonelist = &pgdat\-\>node\_zonelists[0**], 也就是说节点的**备用节点内存域**也是**包含当前节点内存域**的

内核在**build\_zonelists**中按**分配代价**从**昂贵**到**低廉**的次序, **迭代**了**结点中所有的内存域**.

而在build\_zonelists\_node中,则按照分配代价**从低廉到昂贵的次序**, **迭代**了分配**代价不低于当前内存域的内存域(！！！**）.

build\_zonelists\_node函数: 

```cpp
/*
 * Builds allocation fallback zone lists.
 *
 * Add all populated zones of a node to the zonelist.
 */
static int build_zonelists_node(pg_data_t *pgdat, struct zonelist *zonelist, int nr_zones)
{
    struct zone *zone;
    enum zone_type zone_type = MAX_NR_ZONES;

    do {
        zone_type--;
        zone = pgdat->node_zones + zone_type;
        if (populated_zone(zone)) {
            zoneref_set_zone(zone,
                &zonelist->_zonerefs[nr_zones++]);
            check_highest_zone(zone_type);
        }
    } while (zone_type);

    return nr_zones;
}
```

nr\_zones表示从备用列表中的**哪个位置开始填充新项**. 

从**低廉**到**昂贵**的次序**迭代所有内存域**

第一次执行build\_zonelists\_node, 由于列表中尚没有项, 因此调用者传递了nr\_zones为0.

考虑一个系统, 有内存域ZONE\_HIGHMEM、ZONE\_NORMAL、ZONE\_DMA。在第一次运行build\_zonelists\_node时, 实际上会执行下列赋值

```cpp
zonelist->zones[0] = ZONE_HIGHMEM;
zonelist->zones[1] = ZONE_NORMAL;
zonelist->zones[2] = ZONE_DMA;
```

这里实际上是&pgdat\-\>node\_zonelists\[0\]\-\>\_zonerefs[0, 1, 2]的zone设置为这几个

第一个循环依次迭代大于当前结点编号的所有结点

第二个for循环接下来对所有编号小于当前结点的结点生成备用列表项。

### 8.3.2 setup\_pageset初始化per\_cpu缓存

```c
	for_each_possible_cpu(cpu) {
		setup_pageset(&per_cpu(boot_pageset, cpu), 0);
```

最后**遍历每个cpu**, 对每个CPU设置per\-CPU缓存, 即冷热页.

```
struct zone
{
    struct per_cpu_pageset __percpu *pageset;
};
```

前面讲解**内存管理域zone**的时候,提到了**per\-CPU缓存**,即**冷热页**. **pageset是一个指针**, 其容量与系统能够容纳的**CPU的数目的最大值相同(！！！**).

在组织每个节点的zonelist的过程中, **setup\_pageset**初始化了**per\-CPU缓存(冷热页面**)

```cpp
static void setup_pageset(struct per_cpu_pageset *p, unsigned long batch)
{
	pageset_init(p);
	pageset_set_batch(p, batch);
}
```

setup\_pageset()函数入参**p**是一个**struct per\_cpu\_pageset结构体**的指针，**per\_cpu\_pageset结构**是内核的**各个zone**用于**每CPU**的**页面高速缓存管理结构**。该**高速缓存**包含一些**预先分配的页面**，以用于满足**本地CPU**发出的**单一内存页请求**。而struct per\_cpu_pages定义的**pcp**是该**管理结构的成员**，用于**具体页面管理**。这是一个队列, 统一管理冷热页，热页面在队列前面，而冷页面则在队列后面。

在此之前**free\_area\_init\_node初始化内存结点**的时候,内核就**输出了冷热页的一些信息**, 该工作由**zone\_pcp\_init**完成

#### 8.3.2.1 pageset\_init()初始化struct per\_cpu\_pages结构

```c
【file：/mm/page_alloc.c】
static void pageset_init(struct per_cpu_pageset *p)
{
    struct per_cpu_pages *pcp;
    int migratetype;
 
    memset(p, 0, sizeof(*p));
 
    pcp = &p->pcp;
    pcp->count = 0;
    for (migratetype = 0; migratetype < MIGRATE_PCPTYPES; migratetype++)
        INIT_LIST_HEAD(&pcp->lists[migratetype]);
}
```

里面针对**三种迁移类型**初始化了链表. 也就是说**每个内存域**有一个**冷热页数组**(一个CPU一个per\_cpu\_pageset), **每个per\_cpu\_pageset项**有**三个链表**, 每个链表对应一个**迁移类型**.

#### 8.3.2.2 pageset\_set\_batch设置struct per\_cpu\_pages结构

```c
static void pageset_update(struct per_cpu_pages *pcp, unsigned long high,
        unsigned long batch)
{
       /* start with a fail safe value for batch */
    pcp->batch = 1;
    smp_wmb();
 
       /* Update high, then batch, in order */
    pcp->high = high;
    smp_wmb();
 
    pcp->batch = batch;
}
 
/* a companion to pageset_set_high() */
static void pageset_set_batch(struct per_cpu_pageset *p, unsigned long batch)
{
    pageset_update(&p->pcp, 6 * batch, max(1UL, 1 * batch));
}
```

```cpp
void build_all_zonelists(void)
    |---->set_zonelist_order()
         |---->current_zonelist_order = ZONELIST_ORDER_ZONE;
    |
    |---->__build_all_zonelists(NULL);
    |    Memory不支持热插拔, 为每个zone建立后备的zone,
    |    每个zone及自己后备的zone，形成zonelist
    	|
        |---->pg_data_t *pgdat = NULL;
        |     pgdat = &contig_page_data;(单node)
        |
        |---->build_zonelists(pgdat);
        |     为每个zone建立后备zone的列表
            |
            |---->struct zonelist *zonelist = NULL;
            |     enum zone_type j;
            |     zonelist = &pgdat->node_zonelists[0];
            |
            |---->j = build_zonelists_node(pddat, zonelist, 0, MAX_NR_ZONES - 1);
            |     为pgdat->node_zones[0]建立后备的zone，node_zones[0]后备的zone
            |     存储在node_zonelist[0]内，对于node_zone[0]的后备zone，其后备的zone
            |     链表如下(只考虑UMA体系，而且不考虑ZONE_DMA)：
            |     node_zonelist[0]._zonerefs[0].zone = &node_zones[2];
            |     node_zonelist[0]._zonerefs[0].zone_idx = 2;
            |     node_zonelist[0]._zonerefs[1].zone = &node_zones[1];
            |     node_zonelist[0]._zonerefs[1].zone_idx = 1;
            |     node_zonelist[0]._zonerefs[2].zone = &node_zones[0];
            |     node_zonelist[0]._zonerefs[2].zone_idx = 0;
            |     
            |     zonelist->_zonerefs[3].zone = NULL;
            |     zonelist->_zonerefs[3].zone_idx = 0;    
        |
        |---->build_zonelist_cache(pgdat);
              |---->pdat->node_zonelists[0].zlcache_ptr = NULL;
              |     UMA体系结构
              |
        |---->for_each_possible_cpu(cpu)
        |     setup_pageset(&per_cpu(boot_pageset, cpu), 0);
              |详见下文
    |---->vm_total_pages = nr_free_pagecache_pages();
    |    业务：获得所有zone中的present_pages总和.
    |
    |---->page_group_by_mobility_disabled = 0;
    |     对于代码中的判断条件一般不会成立，因为页数会最够多（内存较大）
```

至此，内存管理框架算法基本准备完毕。

# 9 Buddy伙伴算法

start\_kernel()函数接着往下走，下一个函数是mm\_init()：

```c
[init/main.c]
static void __init mm_init(void)
{
	/*
	 * page_ext requires contiguous pages,
	 * bigger than MAX_ORDER unless SPARSEMEM.
	 */
	page_ext_init_flatmem();
	mem_init();
	kmem_cache_init();
	percpu_init_late();
	pgtable_init();
	vmalloc_init();
	ioremap_huge_init();
}
```

**mem\_init**()则是管理**伙伴管理算法的初始化**，此外**kmem\_cache\_init**()是用于**内核slub内存分配体系的初始化**，而**vmalloc\_init**()则是用于**vmalloc的初始化**。

## 9.1 伙伴初始化mem_init()

前面已经分析了linux内存管理算法（伙伴管理算法）的准备工作。

具体的算法初始化是mm\_init()：

```c
[arch/x86/mm/init_64.c]
void __init mem_init(void)
{
	pci_iommu_alloc();

	/* clear_bss() already clear the empty_zero_page */

	register_page_bootmem_info();

	/* this will put all memory onto the freelists */
	free_all_bootmem();
	after_bootmem = 1;

	/* Register memory areas for /proc/kcore */
	kclist_add(&kcore_vsyscall, (void *)VSYSCALL_ADDR,
			 PAGE_SIZE, KCORE_OTHER);

	mem_init_print_info(NULL);
}
```

### 9.1.1 pci\_iommu\_alloc()初始化iommu table表项

pci\_iommu\_alloc()函数主要是将**iommu table先行排序检查**，然后调用**各个表项注册的函数进行初始化**。

### 9.1.2 register\_page\_bootmem\_info()

```c
static void __init register_page_bootmem_info(void)
{
#ifdef CONFIG_NUMA
	int i;

	for_each_online_node(i)
		register_page_bootmem_info_node(NODE_DATA(i));
#endif
}
```

```c
void __init register_page_bootmem_info_node(struct pglist_data *pgdat)
{
	unsigned long i, pfn, end_pfn, nr_pages;
	int node = pgdat->node_id;
	struct page *page;
	struct zone *zone;

	nr_pages = PAGE_ALIGN(sizeof(struct pglist_data)) >> PAGE_SHIFT;
	page = virt_to_page(pgdat);

	for (i = 0; i < nr_pages; i++, page++)
		get_page_bootmem(node, page, NODE_INFO);

	zone = &pgdat->node_zones[0];
	for (; zone < pgdat->node_zones + MAX_NR_ZONES - 1; zone++) {
		if (zone_is_initialized(zone)) {
			nr_pages = zone->wait_table_hash_nr_entries
				* sizeof(wait_queue_head_t);
			nr_pages = PAGE_ALIGN(nr_pages) >> PAGE_SHIFT;
			page = virt_to_page(zone->wait_table);

			for (i = 0; i < nr_pages; i++, page++)
				get_page_bootmem(node, page, NODE_INFO);
		}
	}

	pfn = pgdat->node_start_pfn;
	end_pfn = pgdat_end_pfn(pgdat);

	/* register section info */
	for (; pfn < end_pfn; pfn += PAGES_PER_SECTION) {
		/*
		 * Some platforms can assign the same pfn to multiple nodes - on
		 * node0 as well as nodeN.  To avoid registering a pfn against
		 * multiple nodes we check that this pfn does not already
		 * reside in some other nodes.
		 */
		if (pfn_valid(pfn) && (early_pfn_to_nid(pfn) == node))
			register_page_bootmem_info_section(pfn);
	}
}
```

遍历所有在线节点



### 9.1.3 free\_all\_bootmem()

```c
unsigned long __init free_all_bootmem(void)
{
	unsigned long pages;

	reset_all_zones_managed_pages();

	/*
	 * We need to use NUMA_NO_NODE instead of NODE_DATA(0)->node_id
	 *  because in some case like Node0 doesn't have RAM installed
	 *  low ram will be on Node1
	 */
	pages = free_low_memory_core_early();
	totalram_pages += pages;

	return pages;
}
```

其中reset\_all\_zones\_managed\_pages()是用于**重置管理区zone结构中的managed\_pages成员数据(zone\-\>managed\_pages**)

free\_low\_memory\_core\_early()用于**释放memlock中的空闲以及alloc分配出去的页面并计数**, 对于**系统定义(这个是静态定义的**)的memblock\_**reserved**\_init\_regions和memblock\_**memory**\_init\_regions则仍保留不予以释放。

其中**totalram\_pages**用于记录**内存的总页面数**

free\_low\_memory\_core\_early()实现：

```c
static unsigned long __init free_low_memory_core_early(void)
{
	unsigned long count = 0;
	phys_addr_t start, end;
	u64 i;

	memblock_clear_hotplug(0, -1);

	for_each_reserved_mem_region(i, &start, &end)
		reserve_bootmem_region(start, end);

	for_each_free_mem_range(i, NUMA_NO_NODE, MEMBLOCK_NONE, &start, &end,
				NULL)
		count += __free_memory_core(start, end);

#ifdef CONFIG_ARCH_DISCARD_MEMBLOCK
	{
		phys_addr_t size;

		/* Free memblock.reserved array if it was allocated */
		size = get_allocated_memblock_reserved_regions_info(&start);
		if (size)
			count += __free_memory_core(start, start + size);

		/* Free memblock.memory array if it was allocated */
		size = get_allocated_memblock_memory_regions_info(&start);
		if (size)
			count += __free_memory_core(start, start + size);
	}
#endif

	return count;
}
```

该函数通过for\_each\_free\_mem\_range()**遍历memblock算法中的空闲内存空间(！！！**)，并调用\_\_**free\_memory\_core**()来释放；

而后面的get\_allocated\_memblock\_reserved\_regions\_info()和get\_allocated\_memblock\_memory\_regions\_info()用于获取**通过申请(alloc而得的memblock管理算法空间**，然后**释放**，其中如果其算法管理空间是**系统定义**的memblock\_reserved\_init\_regions和memblock\_memory\_init\_regions则仍**保留不予以释放**。

最终调用的还是\_\_free\_pages()将**页面予以释放**。

将after\_bootmem设为1.

这样就得到**totalram\_pages**是**内存的总页面数**

## 9.2 Buddy算法

伙伴系统是一个结合了**2的方幂个分配器**和**空闲缓冲区合并技术**的内存分配方案, 其基本思想很简单. **内存**被分成**含有很多页面的大块**,**每一块**都是**2的方幂个页面大小**.如果**找不到**想要的块,一个**大块会被分成两部分**,这两部分彼此就成为**伙伴**.其中**一半被用来分配**,而**另一半则空闲**.这些块在以后分配的过程中会继续被二分直至产生一个所需大小的块.当一个块被最终释放时,其伙伴将被检测出来,如果**伙伴也空闲则合并两者**.

- 内核如何记住哪些**内存块是空闲**的

- **分配空闲页面**的方法

- 影响分配器行为的众多标识位

- **内存碎片**的问题和分配器如何处理碎片

### 9.2.1 伙伴系统的结构

#### 9.2.1.1 数据结构

系统内存中的**每个物理内存页（页帧**），都对应于一个**struct page实例**, **每个内存域**都关联了一个struct zone的实例，其中保存了用于**管理伙伴数据的主要数组**

```cpp
//  http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L324
struct zone
{
	 /* free areas of different sizes */
	struct free_area        free_area[MAX_ORDER];
};
```

**struct free\_area**是一个伙伴系统的**辅助数据结构**

```cpp
struct free_area {
	struct list_head        free_list[MIGRATE_TYPES];
    unsigned long           nr_free;
};
```

| 字段 | 描述 |
|:-----:|:-----|
| free\_list | 是用于连接**空闲页()的链表**. 页链表包含**大小相同的连续内存区** |
| nr\_free | 指定了当前内存区中**空闲页块的数目**（对**0阶**内存区的块**逐页计算**，对**1阶内存区**的块**计算2页的数目**，对**2阶内存区**计算**4页集合的数目**，依次类推 |

**每个zone**有一个**MAX\_ORDER(11)大小的数组**, **每个数组项**是**一个数据结构**, 数据结构由**一个空闲页面块链表(每种迁移类型一个**)和**整型数**构成, **链表项**是**连续内存区**, 这个**整型数**表明了这个**链表项数目(空闲块的数目**).

**伙伴系统的分配器**维护**空闲页面(！！！)所组成的块(即内存区**）, 这里**每一块都是2的方幂个页面**, 方幂的指数称为**阶**.

**内存块的长度是2\^order**,其中**order**的范围从**0到MAX\_ORDER**

zone\-\>free\_area[MAX\_ORDER]数组中**阶**作为各个元素的索引,用于指定**对应链表**中的**连续内存区**包含多少个**页帧**.

- 数组中第0个元素的阶为0, 它的free\_list链表域指向具有包含区为单页(2\^0=1)的内存页面链表

- 数组中第1个元素的free\_list域管理的内存区为两页(2^1=2)

- 第2个管理的内存区为4页, 依次类推.

- 直到**2\^{MAX_ORDER-1}个页面大小的块**

![空闲页快](./images/18.png)

基于MAX_ORDER为11的情况，伙伴管理算法**每个页面块链表项**分别包含了：1、2、4、8、16、32、64、128、256、512、1024个连续的页面，**每个页面块**的**第一个页面的物理地址**是**该块大小的整数倍**。假设连续的物理内存，各页面块左右的页面，要么是等同大小，要么就是整数倍，而且还是偶数，形同伙伴。

#### 9.2.1.2 最大阶MAX\_ORDER与FORCE\_MAX\_ZONEORDER配置选项

一般来说**MAX\_ORDER默认定义为11**, 这意味着**一次分配**可以请求的**页数最大是2\^11=2048**个页面

```cpp
[include/linux/mmzone.h]
/* Free memory management - zoned buddy allocator.  */
#ifndef CONFIG_FORCE_MAX_ZONEORDER
#define MAX_ORDER 11
#else
#define MAX_ORDER CONFIG_FORCE_MAX_ZONEORDER
#endif
#define MAX_ORDER_NR_PAGES (1 << (MAX_ORDER - 1))
```

但如果特定于体系结构的代码设置了**FORCE\_MAX\_ZONEORDER**配置选项, 该值也可以手工改变

#### 9.2.1.3 内存区是如何连接的

**每个内存区（每个块）**中**第1页(！！！)内的链表元素**,可用于**将内存区维持在链表**中。因此，也**不必引入新的数据结构（！！！**）来管理**物理上连续的页**，否则这些页不可能在同一内存区中. 如下图所示

![伙伴系统中相互连接的内存区](./images/19.png)

伙伴不必是彼此连接的. 如果**一个内存区**在分配**其间分解为两半**,内核会**自动将未用的一半**加入到**对应的链表**中.

由于内存释放的缘故,**两个内存区都处于空闲状态**,可通过**其地址判断其是否为伙伴**.管理工作较少, 是伙伴系统的一个主要优点.

基于**伙伴系统**的内存管理**专注于某个结点的某个内存域（！！！某个节点的某个内存域！！！**）, **但所有内存域和结点的伙伴系统**都通过**备用分配列表**连接起来.

伙伴系统和内存域／结点之间的关系：

![伙伴系统和内存域／结点之间的关系](./images/20.png)

最后要注意, 有关**伙伴系统**和**当前状态的信息**可以在/**proc/buddyinfo**中获取

x86\_64的16GB系统：

```
[root@tsinghua-pcm ~]# cat /proc/buddyinfo 
Node 0, zone      DMA      0      0      0      0      2      1      1      0      1      1      3
Node 0, zone    DMA32      1      1      1      3      4      3      3      5      0      2    745
Node 0, zone   Normal    166    157    202    437    122    187     77     60     60      8   2003
```

#### 9.2.1.4 传统伙伴系统算法

在**内核分配内存**时,必须记录**页帧的已分配或空闲状态**,以免**两个进程使用同样的内存区域**.由于内存分配和释放非常频繁, 内核还必须保证相关操作尽快完成.内核可以**只分配完整的页帧**.将内存划分为更小的部分的工作, 则委托给**用户空间中的标准库**.标准库将来源于内核的**页帧拆分为小的区域**,并为进程分配内存.

内核中很多时候**要求分配连续页**. 为快速检测内存中的连续区域, 内核采用了一种古老而历经检验的技术: **伙伴系统**

系统中的**空闲内存块（！！！空闲的块**）总是**两两分组**,每组中的**两个内存块称作伙伴**.伙伴的分配可以是彼此独立的. 但如果**两个伙伴都是空闲的**,内核会将其**合并为一个更大的内存块**,作为**下一层次上某个内存块的伙伴**.

如果下一个请求**只需要2个连续页帧**,则由**8页组成的块**会分裂成**2个伙伴**,每个包含**4个页帧**.其中**一块放置回伙伴列表**中，而**另一个**再次分裂成**2个伙伴**,每个包含**2页**。其中**一个回到伙伴系统**，另一个则**传递给应用程序（分配时候会从伙伴系统去掉！！！**）.

在应用程序**释放内存**时,内核可以**直接检查地址**,来判断**是否能够创建一组伙伴**,并合并为一个更大的内存块放回到伙伴列表中,这刚好是**内存块分裂的逆过程（释放内存可能会将内存块放回伙伴列表！！！**）。这提高了较大内存块可用的可能性.

长期运行会导致碎片化问题

### 9.2.2 伙伴算法释放过程

伙伴管理算法的释放过程是，满足条件的**两个页面块**称之为**伙伴**：**两个页面块的大小相同(！！！**)且**两者的物理地址连续(！！！**)。当**某块页面被释放**时，且其**存在空闲的伙伴页面块**，则算法会将其两者**合并为一个大的页面块**，合并后的页面块如果**还可以找到伙伴页面块**，则将会继续**与相邻的块进行合并**，直至到大小为2\^MAX\_ORDER个页面为止。

### 9.2.3 伙伴算法申请过程

伙伴管理算法的申请过程则相反，如果**申请指定大小的页面**在其**页面块链表中不存在**，则会**往高阶的页面块链表进行查找**，如果依旧没找到，则继续往高阶进行查找，**直到找到**为止，否则就是**申请失败**了。如果在**高阶的页面块链表**找到**空闲的页面块**，则会将其**拆分为两块**，如果**拆分后仍比需要的大**，那么**继续拆分**，直至到**大小刚好**为止，这样避免了资源浪费。

### 9.2.4 碎片化问题

在存储管理中

- **内碎片**是指**分配给作业的存储空间**中**未被利用**的部分
- **外碎片**是指系统中无法利用的小存储块.

Linux伙伴系统**分配内存**的大小要求**2的幂指数页**, 这也会产生严重的**内部碎片**.

**伙伴系统**中存的都是**空闲内存块**. 系统长期运行后, 会发生称为**碎片的内存管理问题**。**频繁的分配和释放页帧**可能导致一种情况：系统中有**若干页帧是空闲**的，但却**散布在物理地址空间的各处**。换句话说，系统中**缺乏连续页帧组成的较大的内存块**.

暂且假设内存页面数为60，则长期运行后，其页面的使用情况可能将会如下图（灰色为已分配）。

![config](./images/22.png)

虽然其未被分配的页面仍有25%，但能够申请到的最大页面仅为一页。不过这对用户空间是没有影响的，主要是由于用户态的内存是通过页面映射而得到的。所以不在乎具体的物理页面分布，其仍是可以将其映射为连续的一块内存提供给用户态程序使用。于是用户态可以感知的内存则如下。

![config](./images/23.png)

但是对于内核态，碎片则是个严肃的问题，因为**大部分物理内存**都**直接映射到内核的永久映射区**里面。如果真的存在碎片，将真的如第一张图所示，无法映射到比一页更大的内存，这长期是linux的短板之一。于是为了解决该问题，则引入了反碎片。

目前Linux内核为**解决内存碎片**的方案提供了两类解决方案

- 依据**可移动性组织页**避免内存碎片

- **虚拟可移动内存域**避免内存碎片

#### 9.2.4.1 依据可移动性组织页(页面迁移)

**文件系统也有碎片**，该领域的碎片问题主要通过**碎片合并工具**解决。它们分析文件系统，**重新排序已分配存储块**，从而建立较大的连续存储区.理论上，该方法对物理内存也是可能的，但由于**许多物理内存页不能移动到任意位置**，阻碍了该方法的实施。因此，内核的方法是**反碎片(anti\-fragmentation**),即试图**从最初开始尽可能防止碎片(！！！**).

##### 9.2.4.1.1 反碎片的工作原理

内核将**已分配页**划分为下面3种不同类型。

| 页面类型 | 描述 | 举例 |
|:---------:|:-----|:-----|
| **不可移动页** | 在内存中有**固定位置**, **不能移动**到其他地方. | 核心**内核**分配的**大多数内存**属于该类别 |
| **可移动页** | **可以随意地移动**. | 属于**用户空间应用程序的页**属于该类别. 它们是通过页表映射的<br>如果它们复制到新位置，**页表项可以相应地更新**，应用程序不会注意到任何事 |
| **可回收页** | **不能直接移动, 但可以删除, 其内容可以从某些源重新生成**. | 例如，**映射自文件的数据**属于该类别<br>**kswapd守护进程**会根据可回收页访问的**频繁程度**，周期性释放此类内存.页面回收本身就是一个复杂的过程.内核会在可回收页占据了太多内存时进行回收,在内存短缺(即分配失败)时也可以发起页面回收. |

页的可移动性，依赖该页属于3种类别的哪一种.内核使用的**反碎片技术**,即基于将具有**相同可移动性的页**分组的思想.

为什么这种方法**有助于减少碎片**?

由于**页无法移动**, 导致在原本**几乎全空的内存区**中无法进行**连续分配**. 根据**页的可移动性**, 将其分配到**不同的列表**中, 即可防止这种情形. 例如, **不可移动的页**不能位于**可移动内存区**的中间, 否则就无法从该内存区分配较大的连续内存块.

但要注意, 从**最初开始**,内存**并未划分**为**可移动性不同的区**.这些是在**运行时形成(！！！**)的.

##### 9.2.4.1.2 迁移类型

```cpp
enum {
        MIGRATE_UNMOVABLE,
        MIGRATE_MOVABLE,
        MIGRATE_RECLAIMABLE,
        MIGRATE_PCPTYPES,       /* the number of types on the pcp lists */
        MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA
        MIGRATE_CMA,
#endif
#ifdef CONFIG_MEMORY_ISOLATION
        MIGRATE_ISOLATE,        /* can't allocate from here */
#endif
        MIGRATE_TYPES
};
```

|  宏  | 类型 |
|:----:|:-----|
| MIGRATE\_UNMOVABLE | 不可移动页. 在内存当中有固定的位置，不能移动。内核的核心分配的内存大多属于这种类型； |
| MIGRATE\_MOVABLE | 可移动页. 可以随意移动，用户空间应用程序所用到的页属于该类别。它们通过页表来映射，如果他们复制到新的位置，页表项也会相应的更新，应用程序不会注意到任何改变； |
| MIGRATE\_RECLAIMABLE | 可回收页. 不能直接移动，但可以删除，其内容页可以从其他地方重新生成，例如，映射自文件的数据属于这种类型，针对这种页，内核有专门的页面回收处理； |
| MIGRATE\_PCPTYPES | 是per\_cpu\_pageset,即用来表示**每CPU页框高速缓存**的数据结构中的链表的迁移类型数目 |
| MIGRATE\_HIGHATOMIC |  =MIGRATE\_PCPTYPES,在罕见的情况下，内核需要分配一个高阶的页面块而不能休眠.如果向具有特定可移动性的列表请求分配内存失败，这种紧急情况下可从MIGRATE\_HIGHATOMIC中分配内存 |
| MIGRATE\_CMA | Linux内核最新的**连续内存分配器**(CMA), 用于**避免预留大块内存**. 连续内存分配，用于避免预留大块内存导致系统可用内存减少而实现的，即当驱动不使用内存时，将其分配给用户使用，而需要时则通过回收或者迁移的方式将内存腾出来。 |
| MIGRATE\_ISOLATE | 是一个特殊的虚拟区域,用于**跨越NUMA结点移动物理内存页**.在大型系统上,它有益于将**物理内存页**移动到接近于**使用该页最频繁的CPU**. |
| MIGRATE\_TYPES | 只是表示迁移类型的数目, 也不代表具体的区域 |

对于MIGRATE\_CMA类型, 需要**预留大量连续内存**，这部分内存**平时不用**，但是一般的做法又必须**先预留着**. CMA机制可以做到**不预留内存**，这些内存**平时是可用的**，只有当**需要的时候才被分配**

##### 9.2.4.1.3 可移动性组织页的buddy组织

至于**迁移类型的页面管理**实际上采用的还是**伙伴管理算法的管理方式**，内存管理区zone的结构里面的free\_area是用于管理各阶内存页面，而其里面的**free\_list则是对各迁移类型进行区分的链表**。

```c
struct zone
{
	 /* free areas of different sizes */
	struct free_area        free_area[MAX_ORDER];
};

struct free_area {
	struct list_head        free_list[MIGRATE_TYPES];
    unsigned long           nr_free;
};
```

**每个zone**有一个**MAX\_ORDER(11)大小的数组**, **每个数组项**是**一个数据结构**, 数据结构由**一个空闲页面块链表(每种迁移类型一个**)和**整型数**构成, **链表项**是**连续内存区**, 这个**整型数**表明了这个**链表项数目(空闲块的数目**).

**每种迁移类型**都对应一个**空闲列表**, 内存框图如下

依据可移动性组织页:

![config](./images/24.png)

宏for\_each\_migratetype\_order(order, type)可用于**迭代指定迁移类型的所有分配阶**

```cpp
#define for_each_migratetype_order(order, type) \
        for (order = 0; order < MAX_ORDER; order++) \
                for (type = 0; type < MIGRATE_TYPES; type++)
```

##### 9.2.4.1.4 迁移备用列表fallbacks

内核无法满足针对某一**给定迁移类型**的**分配请求**, 会怎么样?

类似于NUMA的备用内存域列表zonelists. 内存迁移提供了备用列表fallbacks.

```c
[mm/page_alloc.c]
/*
 * This array describes the order lists are fallen back to when
 * the free lists for the desirable migrate type are depleted
 * 该数组描述了指定迁移类型的空闲列表耗尽时
 * 其他空闲列表在备用列表中的次序
 */
static int fallbacks[MIGRATE_TYPES][4] = {
	//  分配不可移动页失败的备用列表
    [MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,   MIGRATE_TYPES },
    //  分配可回收页失败时的备用列表
    [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,   MIGRATE_TYPES },
    //  分配可移动页失败时的备用列表
    [MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_TYPES },
#ifdef CONFIG_CMA
    [MIGRATE_CMA]     = { MIGRATE_TYPES }, /* Never used */
#endif
#ifdef CONFIG_MEMORY_ISOLATION
    [MIGRATE_ISOLATE]     = { MIGRATE_TYPES }, /* Never used */
#endif
};
```

##### 9.2.4.1.5 全局pageblock\_order变量

**页可移动性分组特性**的**全局变量**和**辅助函数**总是**编译到内核**中，但只有在系统中**有足够内存可以分配到多个迁移类型对应的链表（！！！**）时，才是有意义的。

**每个迁移链表(！！！**)都应该有**适当数量的内存(！！！**), 这是通过两个全局变量**pageblock\_order**和**pageblock\_nr\_pages**提供的.

pageblock\_order是一个**大**的分配阶, **pageblock\_nr\_pages**则表示**该分配阶对应的页数**。如果体系结构提供了**巨型页机制**,则**pageblock\_order**通常定义为**巨型页对应的分配阶**.

```cpp
#ifdef CONFIG_HUGETLB_PAGE

    #ifdef CONFIG_HUGETLB_PAGE_SIZE_VARIABLE

        /* Huge page sizes are variable */
        extern unsigned int pageblock_order;

    #else /* CONFIG_HUGETLB_PAGE_SIZE_VARIABLE */

    /* Huge pages are a constant size */
        #define pageblock_order         HUGETLB_PAGE_ORDER

    #endif /* CONFIG_HUGETLB_PAGE_SIZE_VARIABLE */

#else /* CONFIG_HUGETLB_PAGE */

    /* If huge pages are not used, group by MAX_ORDER_NR_PAGES */
    #define pageblock_order         (MAX_ORDER-1)

#endif /* CONFIG_HUGETLB_PAGE */

#define pageblock_nr_pages      (1UL << pageblock_order)
```



如果体系结构**不支持巨型页**, 则将其定义为**第二高的分配阶**, 即MAX\_ORDER \- 1

如果**各迁移类型的链表**中**没有**一块**较大的连续内存**, 那么**页面迁移不会提供任何好处**, 因此在**可用内存太少时内核会关闭该特性**. 这是在**build\_all\_zonelists**函数中检查的, 该函数用于初始化内存域列表.如果**没有足够的内存可用**, 则**全局变量**[**page\_group\_by\_mobility\_disabled**]设置为**0**, 否则设置为1.

内核如何知道**给定的分配内存**属于**何种迁移类型**? 有关**各个内存分配的细节**都通过**分配掩码指定**.

内核提供了两个标志，分别用于表示分配的内存是**可移动**的(**\_\_GFP\_MOVABLE**)或**可回收**的(**\_\_GFP\_RECLAIMABLE**).

##### 9.2.4.1.6 gfpflags\_to\_migratetype转换分配标识到迁移类型

如果标志**都没有设置**,则**分配的内存假定为不可移动的**.

gfpflags\_to\_migratetype函数可用于**转换分配标志及对应的迁移类型**.

```cpp
static inline int gfpflags_to_migratetype(const gfp_t gfp_flags)
{
    VM_WARN_ON((gfp_flags & GFP_MOVABLE_MASK) == GFP_MOVABLE_MASK);
    BUILD_BUG_ON((1UL << GFP_MOVABLE_SHIFT) != ___GFP_MOVABLE);
    BUILD_BUG_ON((___GFP_MOVABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_MOVABLE);

    if (unlikely(page_group_by_mobility_disabled))
        return MIGRATE_UNMOVABLE;

    /* Group based on mobility */
    return (gfp_flags & GFP_MOVABLE_MASK) >> GFP_MOVABLE_SHIFT;
}
```

如果**停用了页面迁移**特性,则**所有的页都是不可移动**的.否则.该函数的**返回值**可以直接用作free\_area.free\_list的**数组索引**.

##### 9.2.4.1.7 内存域zone提供跟踪内存区的属性

**每个内存域**都提供了一个特殊的字段,可以**跟踪包含pageblock\_nr\_pages个页的内存区的属性**. 即zone\-\>pageblock\_flags字段, 当前只有与**页可移动性相关**的代码使用

在**初始化期间**, 内核自动确保对**内存域中**的**每个不同的迁移类型分组**, 在**pageblock\_flags**中都分配了足**够存储NR\_PAGEBLOCK\_BITS个比特位**的空间。当前，表示**一个连续内存区的迁移类型需要3个比特位**

```c
enum pageblock_bits {
    PB_migrate,
    PB_migrate_end = PB_migrate + 3 - 1,
            /* 3 bits required for migrate types */
    PB_migrate_skip,/* If set the block is skipped by compaction */
    NR_PAGEBLOCK_BITS
};
```

内核提供**set\_pageblock\_migratetype**负责设置**以page为首的一个内存区的迁移类型**

```cpp
void set_pageblock_migratetype(struct page *page, int migratetype)
{
    if (unlikely(page_group_by_mobility_disabled &&
             migratetype < MIGRATE_PCPTYPES))
        migratetype = MIGRATE_UNMOVABLE;

    set_pageblock_flags_group(page, (unsigned long)migratetype,
                    PB_migrate, PB_migrate_end);
}
```

**migratetype参数**可以通过上文介绍的**gfpflags\_to\_migratetype辅助函数构建**. 请注意很重要的一点, **页的迁移类型**是**预先分配**好的,对应的比特位总是可用,与**页是否由伙伴系统管理无关**.

在**释放内存**时，页必须返回到**正确的迁移链表**。这之所以可行，是因为能够从**get\_pageblock\_migratetype**获得所需的信息.

```cpp
#define get_pageblock_migratetype(page)                                 \
        get_pfnblock_flags_mask(page, page_to_pfn(page),                \
                        PB_migrate_end, MIGRATETYPE_MASK)
```

##### 9.2.4.1.8 /proc/pagetypeinfo获取页面分配状态

当前的页面分配状态可以从/proc/pagetypeinfo获取

![config](./images/25.png)

##### 9.2.4.1.9 可移动性的分组的初始化

在内存子系统初始化期间, **memmap\_init\_zone**负责处理**内存域的page实例**. 其中会将**所有的页**最初都标记为**可移动**的.

在**内核分配不可移动的内存区**时, 则必须"盗取".

**启动阶段**分配**可移动区**的情况较少, 那么从可移动分区分配内存区, 并将其从可移动列表转换到不可移动列表.

这样避免了**启动期间内核分配的内存**(经常系统**整个运行期间不释放**)散布到物理内存各处, 从而使其他类型的内存分配免受碎片干扰.

#### 9.2.4.2 虚拟可移动内存域

防止物理内存碎片的另一个方法: **虚拟内存域ZONE\_MOVABLE**.

这比可移动性分组更早加入内核. 与可移动性分组相反, ZONE\_MOVABLE特性**必须由管理员显式激活**.

基本思想很简单: **可用的物理内存**划分为**两个内存域**,一个用于**可移动分配**,一个用于**不可移动分配**.这会自动防止不可移动页向可移动内存域引入碎片.

**内核如何在两个竞争的内存域之间分配可用的内存**? 这个问题没办法解决, 所以系统管理员需要抉择.

##### 9.2.4.2.1 数据结构

**kernelcore参数**用来指定用于**不可移动分配的内存数量**,即用于**既不能回收也不能迁移**的内存数量。**剩余的内存用于可移动分配**。在分析该参数之后，结果保存在**全局变量required\_kernelcore**中.

还可以使用参数**movablecore**控制用于**可移动内存分配的内存数量**。**required\_kernelcore**的大小将会据此计算。

同时指定, 分别计算出来required\_kernelcore然后取较大值.

都没有指定, 则该机制无效

```cpp
static unsigned long __initdata required_kernelcore;
static unsigned long __initdata required_movablecore;
```

取决于**体系结构和内核配置**，**ZONE\_MOVABLE内存域**可能位于**高端或普通内存域**, 见

```cpp
enum zone_type {
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

**ZONE\_MOVABLE**并**不关联到任何硬件**上**有意义的内存范围**. 该**内存域中的内存**取自**高端内存域**或**普通内存域**, 因此我们在下文中称ZONE\_MOVABLE是一个虚拟内存域.

辅助函数[**find\_zone\_movable\_pfns\_for\_nodes**]用于**计算进入ZONE\_MOVABLE的内存数量**.

**从物理内存域**提取多少内存用于**ZONE\_MOVABLE**, 必须考虑下面两种情况

- 用于**不可移动分配的内存**会平均地分布到**所有内存结点**上

- **只使用来自最高内存域的内存（！！！**）。在内存较多的**32位系统**上,这**通常会是ZONE\_HIGHMEM**,但是对于**64位系统**，将使用**ZONE\_NORMAL或ZONE\_DMA32**.

计算过程很复杂, 结果是

- 用于为虚拟内存域ZONE\_MOVABLE提取内存页的**物理内存域（先提取内存域**），保存在全局变量**movable\_zone**中

- 对**每个结点**来说, zone\_movable\_pfn[node\_id]表示ZONE\_MOVABLE在**movable\_zone内存域**中所取得**内存的起始地址**.

##### 9.2.4.2.2 实现与应用

类似于页面迁移方法, 分配标志在此扮演了关键角色. 目前只要知道**所有可移动分配**都必须指定\_\_GFP\_HIGHMEM和\_\_GFP\_MOVABLE即可.

由于内核依据分配标志确定进行内存分配的内存域,在**设置了上述的标志**时,可以选择**ZONE\_MOVABLE内存域**.

## 9.3 分配掩码(gfp\_mask标志)

Linux将内存划分为内存域.内核提供了所谓的**内存域修饰符(zone modifier**)(在**掩码的最低4个比特位定义**),来指定从**哪个内存域分配所需的页**.

内核使用宏的方式定义了这些掩码,一个掩码的定义被划分为3个部分进行定义, 共计26个掩码信息, 因此后面\_\_GFP\_BITS\_SHIFT =  26.

### 9.3.1 掩码分类

Linux中这些**掩码标志gfp\_mask**分为3种类型 :

| 类型 | 描述 |
|:-----:|:-----|
| **区描述符(zone modifier**) | 内核把物理内存分为多个区,每个区用于不同的目的,区描述符指明到底从这些区中的**哪一区进行分配** |
| **行为修饰符(action modifier**) | 表示内核应该**如何分配所需的内存**.在某些特定情况下,只能使用某些特定的方法分配内存 |
| 类型标志 | **组合**了**行为修饰符**和**区描述符**,将这些可能用到的组合归纳为**不同类型** |

### 9.3.2 内核中掩码的定义

#### 9.3.2.1 内核中的定义方式

```cpp
//  include/linux/gfp.h

/*  line 12 ~ line 44  第一部分
 *  定义可掩码所在位的信息, 每个掩码对应一位为1
 *  定义形式为  #define	___GFP_XXX		0x01u
 */
/* Plain integer GFP bitmasks. Do not use this directly. */
#define ___GFP_DMA              0x01u
#define ___GFP_HIGHMEM          0x02u
#define ___GFP_DMA32            0x04u
#define ___GFP_MOVABLE          0x08u
/*  ......  */

/*  line 46 ~ line 192  第二部分
 *  定义掩码和MASK信息, 第二部分的某些宏可能是第一部分一个或者几个的组合
 *  定义形式为  #define	__GFP_XXX		 ((__force gfp_t)___GFP_XXX)
 */
#define __GFP_DMA       ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM   ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32     ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE   ((__force gfp_t)___GFP_MOVABLE)  /* ZONE_MOVABLE allowed */
#define GFP_ZONEMASK    (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)

/*  line 194 ~ line 260  第三部分
 *  定义掩码
 *  定义形式为  #define	GFP_XXX		 __GFP_XXX
 */
#define GFP_DMA         __GFP_DMA
#define GFP_DMA32       __GFP_DMA32
```

其中**GFP**缩写的意思为**获取空闲页(get free page**), \_\_GFP\_MOVABLE不表示物理内存域, 但通知内核应在特殊的虚拟内存域ZONE\_MOVABLE进行相应的分配.

#### 9.3.2.2 定义掩码位

看第一部分, 一共**26个掩码信息**

```cpp
/* Plain integer GFP bitmasks. Do not use this directly. */
//  区域修饰符
#define ___GFP_DMA              0x01u
#define ___GFP_HIGHMEM          0x02u
#define ___GFP_DMA32            0x04u

//  行为修饰符
#define ___GFP_MOVABLE          0x08u	    /* 页是可移动的 */
#define ___GFP_RECLAIMABLE      0x10u	    /* 页是可回收的 */
#define ___GFP_HIGH             0x20u		/* 应该访问紧急分配池？ */
#define ___GFP_IO               0x40u		/* 可以启动物理IO？ */
#define ___GFP_FS               0x80u		/* 可以调用底层文件系统？ */
#define ___GFP_COLD             0x100u	   /* 需要非缓存的冷页 */
#define ___GFP_NOWARN           0x200u	   /* 禁止分配失败警告 */
#define ___GFP_REPEAT           0x400u	   /* 重试分配，可能失败 */
#define ___GFP_NOFAIL           0x800u	   /* 一直重试，不会失败 */
#define ___GFP_NORETRY          0x1000u	  /* 不重试，可能失败 */
#define ___GFP_MEMALLOC         0x2000u  	/* 使用紧急分配链表 */
#define ___GFP_COMP             0x4000u	  /* 增加复合页元数据 */
#define ___GFP_ZERO             0x8000u	  /* 成功则返回填充字节0的页 */
//  类型修饰符
#define ___GFP_NOMEMALLOC       0x10000u	 /* 不使用紧急分配链表 */
#define ___GFP_HARDWALL         0x20000u	 /* 只允许在进程允许运行的CPU所关联的结点分配内存 */
#define ___GFP_THISNODE         0x40000u	 /* 没有备用结点，没有策略 */
#define ___GFP_ATOMIC           0x80000u 	/* 用于原子分配，在任何情况下都不能中断  */
#define ___GFP_ACCOUNT          0x100000u
#define ___GFP_NOTRACK          0x200000u
#define ___GFP_DIRECT_RECLAIM   0x400000u
#define ___GFP_OTHER_NODE       0x800000u
#define ___GFP_WRITE            0x1000000u
#define ___GFP_KSWAPD_RECLAIM   0x2000000u
```

#### 9.3.2.3 定义掩码位

..............

## 9.3 alloc\_pages分配内存空间

前面只是大概描述了伙伴系统分配页面过程, 下面详细看

伙伴管理算法内存申请和释放的入口一样，其实并没有很清楚的界限表示这个函数是入口，而那个不是. 所有函数有一个共同点: **只能分配2的整数幂个连续的页**.

![config](./images/26.png)

所有函数最终会统一到alloc\_pages()宏定义入口, 另外所有体系结构都必须实现clear\_page, 可帮助alloc\_pages对页填充字节0

```c
[include/linux/gfp.h]
#define alloc_pages(gfp_mask, order) \
        alloc_pages_node(numa_node_id(), gfp_mask, order)
```

```c
[/include/linux/gfp.h]
static inline struct page *alloc_pages_node(int nid, gfp_t gfp_mask,
                        unsigned int order)
{
    /* Unknown node is current node */
    if (nid < 0)
        nid = numa_node_id();
 
    return __alloc_pages(gfp_mask, order, node_zonelist(nid, gfp_mask));
}

static inline struct zonelist *node_zonelist(int nid, gfp_t flags)
{
	return NODE_DATA(nid)->node_zonelists + gfp_zonelist(flags);
}

static inline int gfp_zonelist(gfp_t flags)
{
#ifdef CONFIG_NUMA
	if (unlikely(flags & __GFP_THISNODE))
		return ZONELIST_NOFALLBACK;
#endif
	return ZONELIST_FALLBACK;
}
```

**没有**明确内存申请的**node节点**时，则**默认**会选择**当前的node**节点作为申请节点。

调用\_\_**alloc\_pages**()来申请具体内存，其中**入参node\_zonelist**()是用于获取**node节点的zone管理区列表(备用节点列表**)。

```c
[/include/linux/gfp.h]
static inline struct page *
__alloc_pages(gfp_t gfp_mask, unsigned int order,
        struct zonelist *zonelist)
{
    return __alloc_pages_nodemask(gfp_mask, order, zonelist, NULL);
}
```

内核源代码将\_\_**alloc\_pages\_nodemask**称之为"伙伴系统的心脏", 因为它处理的是**实质性的内存分配**.

我们先转向页面选择是如何工作的

### 9.3.1 页面选择

#### 9.3.1.1 内存水位标志

```c
enum zone_watermarks {
        WMARK_MIN,
        WMARK_LOW,
        WMARK_HIGH,
        NR_WMARK
};

#define min_wmark_pages(z) (z->watermark[WMARK_MIN])
#define low_wmark_pages(z) (z->watermark[WMARK_LOW])
#define high_wmark_pages(z) (z->watermark[WMARK_HIGH])
```

内核需要定义一些**函数使用**的**标志**, 用于控制到达**各个水位**指定的临界状态时的**行为**, 这些标志用宏来定义

```c
/* The ALLOC_WMARK bits are used as an index to zone->watermark */
#define ALLOC_WMARK_MIN         WMARK_MIN	/*  1 = 0x01, 使用pages_min水印  */
#define ALLOC_WMARK_LOW         WMARK_LOW	/*  2 = 0x02, 使用pages_low水印  */
#define ALLOC_WMARK_HIGH        WMARK_HIGH   /*  3 = 0x03, 使用pages_high水印  */
#define ALLOC_NO_WATERMARKS     0x04 /* don't check watermarks at all  完全不检查水印 */

/* Mask to get the watermark bits */
#define ALLOC_WMARK_MASK        (ALLOC_NO_WATERMARKS-1)

#define ALLOC_HARDER            0x10 /* try to alloc harder, 试图更努力地分配, 即放宽限制  */
#define ALLOC_HIGH              0x20 /* __GFP_HIGH set, 设置了__GFP_HIGH */
#define ALLOC_CPUSET            0x40 /* check for correct cpuset, 检查内存结点是否对应着指定的CPU集合 */
#define ALLOC_CMA               0x80 /* allow allocations from CMA areas */
#define ALLOC_FAIR              0x100 /* fair zone allocation */
```

前几个标志(**ALLOC\_WMARK\_MIN**, ALLOC\_WMARK\_**LOW**, ALLOC\_WMARK\_**HIGH**, ALLOC\_**NO**\_WATERMARKS)表示在**判断页是否可分配时**, 需要考虑**哪些水印**. **默认**情况下(即没有因其他因素带来的压力而需要更多的内存), 只有**内存域包含页的数目**至少为**zone\-\>pages\_high**时, **才能分配页**.这对应于ALLOC\_WMARK\_**HIGH**标志. 如果要使用较低(zone\-\>pages\_**low**)或最低(zone\-\>pages\_**min**)设置, 则必须**相应地设置**ALLOC\_WMARK\_MIN或ALLOC\_WMARK\_LOW. 而ALLOC\_**NO**\_WATERMARKS则通知内核在**进行内存分配**时**不要考虑内存水印**.

ALLOC\_HARDER通知伙伴系统在急需内存时**放宽分配规则**. 在分配高端内存域的内存时, ALLOC\_HIGH进一步放宽限制.

ALLOC\_CPUSET告知内核, 内存只能从当前进程允许运行的**CPU相关联的内存结点分配**, 当然该选项**只对NUMA系统有意义**.

ALLOC\_CMA通知伙伴系统从**CMA区域**中**分配内存**

最后, ALLOC\_FAIR则希望内核公平(均匀)的从内存域zone中进行内存分配

#### 9.3.1.2 zone\_watermark\_ok函数检查标志

设置的标志在zone\_watermark\_ok函数中检查, 该函数根据**设置的标志**判断是否能从**给定的内存域(！！！**)中**分配内存**.

```c
bool zone_watermark_ok(struct zone *z, unsigned int order, unsigned long mark,
              int classzone_idx, unsigned int alloc_flags)
{
    return __zone_watermark_ok(z, order, mark, classzone_idx, alloc_flags,
                    zone_page_state(z, NR_FREE_PAGES));
}
```

- zone\_page\_state访问每个内存域的统计量, 然后得到**空闲页的数目**

- 根据**标志**设置**最小值标记值**

- 检查**空闲页数目**是否小于等于**最小值**与**lowmem\_reserve(zone\-\>lowmem\_reserve[zone**]. 这个是为**各种内存域指定的若干页**, 用于一些**无论如何不能失败的关键性内存访问**)中指定的**紧急分配值min**之和, 是的话返回false

- 如果**不小于**, 遍历所有**大于等于当前阶的分配阶**, 其中z\-\>free\_area\[阶数\]\-\>nr\_free是**当前分配阶**的**空闲块的数目**, `struct free_area *area = &z->free_area[阶数]`, 遍历这个area的MIGRATE\_UNMOVABLE(不可移动页), MIGRATE\_MOVABLE(可移动页), MIGRATE\_RECLAIMABLE(可回收页)看这三个链表是否为空, 都为空, 则不进行内存分配.

- 不符合要求返回false

#### 9.3.1.3 get\_page\_from\_freelist实际分配

通过**标志集和分配阶来判断是否能进行分配**。如果可以，则发起**实际的分配操作**.

该函数随着不断演变, 支持的特性很多, 参数很复杂, 所以后面讲那些相关联的参数封装成一个结构

```c
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags, const struct alloc_context *ac)
```

这个结构是struct alloc\_context

```c
struct alloc_context {
        struct zonelist *zonelist;
        nodemask_t *nodemask;
        struct zoneref *preferred_zoneref;
        int migratetype;
        enum zone_type high_zoneidx;
        bool spread_dirty_pages;
};
```

| 字段 | 描述 |
|:-----:|:-----|
| zonelist | 当**perferred\_zone**上**没有合适的页**可以分配时，就要**按zonelist中的顺序**扫描该zonelist中**备用zone列表**，一个个的**试用** |
| nodemask | 表示**节点的mask**，就是**是否能在该节点上分配内存**，这是个**bit位数组** |
| **preferred\_zoneref** | 表示从**high\_zoneidx**后找到的**合适的zone**，一般会从**该zone分配**；**分配失败**的话，就会在**zonelist**再找一个**preferred\_zone = 合适的zone** |
| migratetype | **迁移类型**，在**zone\-\>free\_area.free\_list[XXX**] 作为**分配下标**使用，这个是用来**反碎片化**的，修改了以前的free\_area结构体，在该结构体中再添加了一个数组，该数组以迁移类型为下标，每个数组元素都挂了对应迁移类型的页链表 |
| **high\_zoneidx** | 是表示该分配时，**所能分配的最高zone**，一般从high-->normal-->dma 内存越来越昂贵，所以一般从high到dma分配依次分配 |
| spread\_dirty\_pages | |

**zonelist是指向备用列表的指针**. 在**预期内存域(！！！)没有空闲空间**的情况下, 该列表确定了**扫描系统其他内存域(和结点)的顺序**.

随后for循环**遍历备用列表的所有内存域**，用最简单的方式查找**一个适当的空闲内存块**

- 首先，解释ALLOC\_\*标志(\_\_cpuset\_zone\_allowed\_softwall是另一个辅助函数, 用于**检查给定内存域是否属于该进程允许运行的CPU**).

- zone\_watermark\_ok接下来**检查所遍历到的内存域是否有足够的空闲页**，并**试图分配一个连续内存块**。如果**两个条件之一**不能满足，即或者**没有足够的空闲页**，或者**没有连续内存块**可满足分配请求，则循环进行到**备用列表中的下一个内存域**，作同样的检查. 直到找到一个合适的页面, 去**try\_this\_zone(一个标记)进行内存分配**

- 如果**内存域适用于当前的分配请求**, 那么**buffered\_rmqueue**试图从中**分配所需数目的页**

```c
static inline
struct page *buffered_rmqueue(struct zone *preferred_zone,
            struct zone *zone, int order, gfp_t gfp_flags,
            int migratetype)
```

**buffered\_rmqueue**是真正的用于分配内存页面的函数

```c
	if (likely(order == 0)) {
		struct per_cpu_pages *pcp;
		struct list_head *list;

		local_irq_save(flags);
		do {
			pcp = &this_cpu_ptr(zone->pageset)->pcp;
			list = &pcp->lists[migratetype];
			if (list_empty(list)) {
				pcp->count += rmqueue_bulk(zone, 0,
						pcp->batch, list,
						migratetype, cold);
				if (unlikely(list_empty(list)))
					goto failed;
			}

			if (cold)
				page = list_last_entry(list, struct page, lru);
			else
				page = list_first_entry(list, struct page, lru);

			__dec_zone_state(zone, NR_ALLOC_BATCH);
			list_del(&page->lru);
			pcp->count--;

		} while (check_new_pcp(page));
	}
```

- 如果**申请的内存页面**处于伙伴算法中的**0阶**, 即**只申请一个内存页面**, 则首先尝试从**冷热页**中**申请**, **申请失败**继而调用**rmqueue\_bulk**()**申请页面**至**冷热页管理列表(也就是申请一个页面并将其加入冷热页管理列表**)中, 继而再从**冷热页列表中获取**; 如果申请**多个页面**则通过\_\_rmqueue()直接从**伙伴管理**中申请.
- \_\_rmqueue()直接从**伙伴管理**中申请
   
```c
【file:/mm/page_alloc.c】
/*
 * Do the hard work of removing an element from the buddy allocator.
 * Call me with the zone->lock already held.
 */
static struct page *__rmqueue(struct zone *zone, unsigned int order,
                        int migratetype)
{
	struct page *page;

	page = __rmqueue_smallest(zone, order, migratetype);
	if (unlikely(!page)) {
		if (migratetype == MIGRATE_MOVABLE)
			page = __rmqueue_cma_fallback(zone, order);

		if (!page)
			page = __rmqueue_fallback(zone, order, migratetype);
	}

	trace_mm_page_alloc_zone_locked(page, order, migratetype);
	return page;
}
```
    
- \_\_rmqueue\_smallest: 分配算法的核心功能. 从指定的order阶开始循环, 如果**该阶的链表不为空**，则直接通过**list\_del**()从**该链表**中**获取空闲页面**以满足申请需要；如果**该阶的链表为空**，则往**更高一阶的链表查找**，直到找到链表不为空的一阶，至于若找到了**最高阶仍为空链表**，则**申请失败**；否则将在找到**链表不为空的一阶**后，将**空闲页面块通过list\_del**()从**链表中摘除**出来，然后通过**expand**()将其**对等拆分开**，并将**拆分出来的一半空闲**部分**挂接至低一阶的链表**中，直到**拆分至恰好满足申请需要的order阶**，最后将得到的满足要求的页面返回回去。至此，页面已经分配到了。
- \_\_rmqueue\_fallback: 其主要是向**其他迁移类型中获取内存**。较正常的伙伴算法不同，其向迁移类型的内存申请内存页面时，是从**最高阶开始查找**的，主要是从大块内存中申请可以避免更少的碎片。


### 9.3.2 伙伴系统核心\_\_alloc\_pages\_nodemask实质性的内存分配

\_\_alloc\_pages\_nodemask是伙伴系统的心脏

```c
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
			struct zonelist *zonelist, nodemask_t *nodemask)
{
	struct page *page;
	unsigned int cpuset_mems_cookie;
	unsigned int alloc_flags = ALLOC_WMARK_LOW|ALLOC_FAIR;
	gfp_t alloc_mask = gfp_mask; /* The gfp_t that was actually used for allocation */
	struct alloc_context ac = {
		.high_zoneidx = gfp_zone(gfp_mask),
		.zonelist = zonelist,
		.nodemask = nodemask,
		.migratetype = gfpflags_to_migratetype(gfp_mask),
	};

	if (cpusets_enabled()) {
		alloc_mask |= __GFP_HARDWALL;
		alloc_flags |= ALLOC_CPUSET;
		if (!ac.nodemask)
			ac.nodemask = &cpuset_current_mems_allowed;
	}

	gfp_mask &= gfp_allowed_mask;

	lockdep_trace_alloc(gfp_mask);

	might_sleep_if(gfp_mask & __GFP_DIRECT_RECLAIM);

	if (should_fail_alloc_page(gfp_mask, order))
		return NULL;

	/*
	 * Check the zones suitable for the gfp_mask contain at least one
	 * valid zone. It's possible to have an empty zonelist as a result
	 * of __GFP_THISNODE and a memoryless node
	 */
	if (unlikely(!zonelist->_zonerefs->zone))
		return NULL;

	if (IS_ENABLED(CONFIG_CMA) && ac.migratetype == MIGRATE_MOVABLE)
		alloc_flags |= ALLOC_CMA;

retry_cpuset:
	cpuset_mems_cookie = read_mems_allowed_begin();

	/* Dirty zone balancing only done in the fast path */
	ac.spread_dirty_pages = (gfp_mask & __GFP_WRITE);

	/*
	 * The preferred zone is used for statistics but crucially it is
	 * also used as the starting point for the zonelist iterator. It
	 * may get reset for allocations that ignore memory policies.
	 */
	ac.preferred_zoneref = first_zones_zonelist(ac.zonelist,
					ac.high_zoneidx, ac.nodemask);
	if (!ac.preferred_zoneref) {
		page = NULL;
		goto no_zone;
	}

	/* First allocation attempt */
	page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
	if (likely(page))
		goto out;

	/*
	 * Runtime PM, block IO and its error handling path can deadlock
	 * because I/O on the device might not complete.
	 */
	alloc_mask = memalloc_noio_flags(gfp_mask);
	ac.spread_dirty_pages = false;

	/*
	 * Restore the original nodemask if it was potentially replaced with
	 * &cpuset_current_mems_allowed to optimize the fast-path attempt.
	 */
	if (cpusets_enabled())
		ac.nodemask = nodemask;
	page = __alloc_pages_slowpath(alloc_mask, order, &ac);

no_zone:
	/*
	 * When updating a task's mems_allowed, it is possible to race with
	 * parallel threads in such a way that an allocation can fail while
	 * the mask is being updated. If a page allocation is about to fail,
	 * check if the cpuset changed during allocation and if so, retry.
	 */
	if (unlikely(!page && read_mems_allowed_retry(cpuset_mems_cookie))) {
		alloc_mask = gfp_mask;
		goto retry_cpuset;
	}

out:
	if (kmemcheck_enabled && page)
		kmemcheck_pagealloc_alloc(page, order, gfp_mask);

	trace_mm_page_alloc(page, order, alloc_mask, ac.migratetype);

	return page;
}
EXPORT_SYMBOL(__alloc_pages_nodemask);
```

gfpflags\_to\_migratetype(gfp\_mask)转换申请页面的类型为**迁移类型**.

如果申请页面传入的**gfp\_mask掩码**携带\_\_GFP\_DIRECT\_RECLAIM标识，表示允许页面申请时休眠，则会进入might\_sleep\_if()检查是否需要休眠等待以及重新调度

if (unlikely(!zonelist\-\>\_zonerefs\-\>zone))用于检查**当前申请页面的内存管理区zone是否为空**；

read\_mems\_allowed\_begin()用于获得当前对被顺序计数保护的共享资源进行读访问的顺序号，用于避免并发的情况下引起的失败

**first\_zones\_zonelist**()则是用于根据**nodemask**，找到合适的**不大于high\_zoneidx**的**内存管理区preferred\_zone**；

最后分配内存页面的关键函数**get\_page\_from\_freelist**()和\_\_**alloc\_pages\_slowpath**()

**get\_page\_from\_freelist**()最先尝试页面分配, 如果分配失败, 则进一步调用\_\_**alloc\_pages\_slowpath**(), 用于**慢速页面分配**, 允许等待和内存回收. \_\_**alloc\_pages\_slowpath**()涉及其他内存机制(**内存溢出保护机制OOM**), 后续说明.

## 9.4 释放内存空间

有4个函数用于释放不再使用的页，与所述函数稍有不同

| 内存释放函数 | 描述 |
|:--------------|:-----|
| [free\_page(struct page *)](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L520)<br>[free\_pages(struct page *, order)](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L3918) | 用于将**一个**或**2\^order**页返回给内存管理子系统。内存区的**起始地址**由指向该内存区的**第一个page实例的指针表示** |
| [\_\_free\_page(addr)](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L519)<br>[\_\_free\_pages(addr, order)](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L3906) | 类似于前两个函数，但在表示需要释放的内存区时，使用了**虚拟内存地址**而不是page实例 |

free\_pages是通过\_\_free\_pages来完成内存释放的

```c
void free_pages(unsigned long addr, unsigned int order)
{
    if (addr != 0) {
        VM_BUG_ON(!virt_addr_valid((void *)addr));
        __free_pages(virt_to_page((void *)addr), order);
    }
}
```

首先必须将虚拟地址转换为指向struct page的指针

virt\_to\_page将虚拟内存地址转换为指向page实例的指针. 基本上, 这是讲解内存分配函数时介绍的page\_address辅助函数的逆过程.

各个内存释放函数之间的关系

![config](./images/27.png)

```c
void __free_pages(struct page *page, unsigned int order)
{
    if (put_page_testzero(page)) {
        if (order == 0)
            free_hot_cold_page(page, 0);
        else
            __free_pages_ok(page, order);
    }
}
```

\_\_free\_pages()函数会分两种情况，对于order等于0的情况，做特殊处理；对于order大于0 的情况，属于正常处理流程。

put\_page\_testzero()是对page结构的\_count引用计数做**原子减及测试**，用于**检查内存页面是否仍被使用**，如果不再使用，则进行释放。

其中**order**表示**页面数量**，如果释放的是单页，则会调用free\_hot\_cold\_page()将**页面释放**至**per\-cpu page缓存**中，而**不是伙伴管理算法**；真正的**释放至伙伴管理算法**的是\_\_free\_pages\_ok()，同时也是用于多个页面释放的情况。

### 9.4.1 free\_hot\_cold\_page()释放至per\-cpu缓存(冷热页)

检查页面是否允许释放

get\_pfnblock\_migratetype()和set\_freepage\_migratetype()分别是获取和设置**页面的迁移类型**，即设置到page\-\>index；

**local\_irq\_save**()和末尾的**local\_irq\_restore**()则用于**保存恢复中断请求标识**。

如果**某个页面类型大于MIGRATE\_PCPTYPES**则表示其可挂到**可移动列表**中，如果**迁移类型**是**MIGRATE\_ISOLATE**则直接将该其**释放到伙伴管理算法**中, 结束。 

pcp = \&this\_cpu\_ptr(zone\-\>pageset)\-\>pcp;得到内存管理区(zone)的Per\-CPU管理结构(即冷热页), **冷页**就将其(\&page\-\>lru)挂接到**对应迁移类型**的**链表尾**，而若是**热页**则挂接到**对应迁移类型的链表头**。

如果**Per\-CPU缓存的页面数目**超过了**Per\-CPU缓存的最大页面数水位high**时，则将**其批量释放至伙伴管理算法**中(**free\_pcppages\_bulk**(zone, batch, pcp);)，其中**批量数为pcp\->batch**。

#### 9.4.1.1 free\_pcppages\_bulk()释放至伙伴管理算法

```c
static void free_pcppages_bulk(struct zone *zone, int count,
					struct per_cpu_pages *pcp)
{
	int migratetype = 0;
	int batch_free = 0;
	unsigned long nr_scanned;
	bool isolated_pageblocks;

	spin_lock(&zone->lock);
	isolated_pageblocks = has_isolate_pageblock(zone);
	nr_scanned = zone_page_state(zone, NR_PAGES_SCANNED);
	if (nr_scanned)
		__mod_zone_page_state(zone, NR_PAGES_SCANNED, -nr_scanned);

	while (count) {
		struct page *page;
		struct list_head *list;

		/*
		 * Remove pages from lists in a round-robin fashion. A
		 * batch_free count is maintained that is incremented when an
		 * empty list is encountered.  This is so more pages are freed
		 * off fuller lists instead of spinning excessively around empty
		 * lists
		 */
		do {
			batch_free++;
			if (++migratetype == MIGRATE_PCPTYPES)
				migratetype = 0;
			list = &pcp->lists[migratetype];
		} while (list_empty(list));

		/* This is the only non-empty list. Free them all. */
		if (batch_free == MIGRATE_PCPTYPES)
			batch_free = count;

		do {
			int mt;	/* migratetype of the to-be-freed page */

			page = list_last_entry(list, struct page, lru);
			/* must delete as __free_one_page list manipulates */
			list_del(&page->lru);

			mt = get_pcppage_migratetype(page);
			/* MIGRATE_ISOLATE page should not go to pcplists */
			VM_BUG_ON_PAGE(is_migrate_isolate(mt), page);
			/* Pageblock could have been isolated meanwhile */
			if (unlikely(isolated_pageblocks))
				mt = get_pageblock_migratetype(page);

			if (bulkfree_pcp_prepare(page))
				continue;

			__free_one_page(page, page_to_pfn(page), zone, 0, mt);
			trace_mm_page_pcpu_drain(page, 0, mt);
		} while (--count && --batch_free && !list_empty(list));
	}
	spin_unlock(&zone->lock);
}
```

while大循环用于计数释放**指定批量数**的**页面**. 其中释放方式是先自**MIGRATE\_UNMOVABLE**迁移类型起（**止于MIGRATE\_PCPTYPES迁移类型**），遍历各个链表统计其链表中页面数

```c
        do {
            batch_free++;
            if (++migratetype == MIGRATE_PCPTYPES)
                migratetype = 0;
            list = &pcp->lists[migratetype];
        } while (list_empty(list));
```

如果**只有MIGRATE\_PCPTYPES迁移类型**的链表**为非空链表**，则**全部页面**将从**该链表中释放**。

后面的do{}while()里面，其先**将页面从lru链表中去除**，然后**获取页面的迁移类型**，通过\_\_**free\_one\_page()释放页面**。

##### 9.4.1.1.1 \_\_free\_one\_page()释放页面

对释放的页面进行检查校验操作。

while (order \< MAX\_ORDER\-1)循环, 通过\_\_find\_buddy\_index()获取**与当前释放的页面**处于**同一阶的伙伴页面索引值**，同时藉此**索引值**计算出**伙伴页面地址**，并做**伙伴页面检查**以确定其**是否可以合并**，若否则退出；接着if (page\_is\_guard(buddy))用于对页面的debug\_flags成员做检查，由于未配置CONFIG\_DEBUG\_PAGEALLOC，page\_is\_guard()固定返回false；则剩下的操作主要就是**将页面**从**分配链中摘除**，同时**将页面合并**并**将其处于的阶提升一级**。

退出while循环后，通过**set\_page\_order**()设置页面**最终可合并成为的管理阶**。最后判断**当前合并的页面**是否为**最大阶**，**否则**将**页面**放至**伙伴管理链表的末尾**，**避免其过早被分配**，得以机会进一步**与高阶页面进行合并**。末了，将最后的**挂入的阶的空闲计数加1**。

至此伙伴管理算法的页面释放完毕。

### 9.4.2 \_\_free\_pages\_ok释放至伙伴管理算法

调用栈是：

```c
__free_pages_ok()

—>free_one_page()

—>__free_one_page()
```

殊途同归，最终还是\_\_free\_one\_page()来释放

# 10 连续内存分配器(CMA)

CMA（**Contiguous Memory Allocator**，连续内存分配器）是在内核3.5的版本引入，由三星的工程师开发实现的，用于**DMA映射框架**下提升**连续大块内存的申请**。

其实现主要是在**系统引导时获取内存**，并将内存设置为**MIGRATE\_CMA迁移类型**，然后再**将内存归还给系统**。内核分配内存时，在**CMA管理内存**中**仅允许**申请**可移动类型内存页面（movable pages**），例如**DMA映射时不使用的页面缓存**。而通过dma\_alloc\_from\_contiguous()申请**大块连续内存**时，将会把这些**可移动页面从CMA管理区中迁移出去**，以便腾出足够的连续内存空间满足申请需要。由此，实现了**任何时刻**只要系统中有**足够的内存空间**，便可以申请得到大块连续内存。

# 11 内存溢出保护机制(OOM)

Linux系统内存管理中存在着一个称之为OOM killer（Out\-Of\-Memory killer）的机制，该机制主要用于**内存监控**，监控**进程的内存使用量**，当**系统的内存耗尽**时，其将根据算法**选择性地kill了部分进程**。本文分析的内存溢出保护机制，也就是OOM killer机制了。

伙伴管理算法中涉及的一函数\_\_alloc\_pages\_nodemask()，其里面调用的\_\_alloc\_pages\_slowpath()并未展开深入，而**内存溢出保护机制**则在此函数中。

首先判断**调用者是否禁止唤醒kswapd线程(每个node有一个**)，若**不做禁止**则**唤醒线程**进行**内存回收**工作

通过**gfp\_to\_alloc\_flags**()得到**内存分配标识**

调用**get\_page\_from\_freelist**()尝试**分配**，如果**分配到则退出**。

判断是否设置了**ALLOC\_NO\_WATERMARKS标识**，如果**设置**了，则将**忽略watermark**，调用\_\_**alloc\_pages\_high\_priority**()进行分配。其中do{}while循环(前提是如果**分配标识有\_\_GFP\_NOFAIL**)不断调用**get\_page\_from\_freelist**()**尝试去获得内存**, 直到分配成功然后退出.

判断是否设置了\_\_**GFP\_WAIT**标识，如果设置则表示**内存分配运行休眠**，否则直接**以分配内存失败而退出**。

调用\_\_**alloc\_pages\_direct\_compact**()和\_\_**alloc\_pages\_direct\_reclaim**()尝试**回收内存并尝试分配**。

基于上面的**多种尝试内存分配仍然失败**的情况，将会调用\_\_**alloc\_pages\_may\_oom**()触发**OOM killer机制(！！！**)。OOM killer将**进程kill后**会重新**再次尝试内存分配**，最后则是**分配失败**或**分配成功**的收尾处理。

## 11.1 \_\_alloc\_pages\_may\_oom()触发OOM killer机制

首先通过**try\_set\_zonelist\_oom**()判断OOM killer是否已经在**其他核进行killing操作**，如果**没有**的情况下将会在**try\_set\_zonelist\_oom**()内部进行**锁操作**，确保**只有一个核执行killing的操作**。

调用**get\_page\_from\_freelist**()在**高watermark**的情况下尝试**再次获取内存**，不过这里**注定会失败**。

调用到**关键函数out\_of\_memory**()。

最后函数退出时将会调用**clear\_zonelist\_oom**()**清除**掉try\_set\_zonelist\_oom()里面的**锁操作**。

### 11.1.1 out\_of\_memory()

首先调用**blocking\_notifier\_call\_chain**()进行**OOM的内核通知链回调处理**；

接着的if (fatal\_signal\_pending(current) || current\-\>flags \& PF\_EXITING)判断则是用于检查**是否有SIGKILL信号挂起或者正在信号处理**中，如果有则退出；

再接着通过**constrained\_alloc**()检查**内存分配限制**以及check\_panic\_on\_oom()检查**是否报linux内核panic**；

继而判断**sysctl\_oom\_kill\_allocating\_task**变量及进程检查，如果符合条件判断，则**将当前分配的进程kill掉**；否则最后，将通过**select\_bad\_process**()选出**最佳的进程**，进而调用**oom\_kill\_process**()对其**进行kill操作**。

#### 11.1.1.1 select\_bad\_process()选择进程

通过for\_each\_process\_thread()宏**遍历所有进程**，进而借用**oom\_scan\_process\_thread**()获得**进程扫描类型**然后通过**switch\-case作特殊化**处理，例如存在**某进程退出中则中断扫描**、**某进程占用内存过多且被标识为优先kill掉则优选**等特殊处理。而**正常情况**则会通过**oom\_badness**()计算出**进程的分值**，然后根据**最高分值将进程控制块返回**回去。

**oom\_badness**()计算出**进程的分值**:

计算进程分值的函数中，首先**排除了不可OOM kill的进程**以及**oom\_score\_adj值((long)task\->signal\-\>oom\_score\_adj**)为**OOM\_SCORE\_ADJ\_MIN（即\-1000）的进程**，其中**oom\_score\_adj取值范围**是\-**1000到1000**；

接着就是**计算进程的RSS**、**页表**以及**SWAP空间**的**使用量**占RAM的**比重**，如果该进程是**超级进程**，则**去除3%的权重**；

最后将oom\_score\_adj和points归一后，但凡**小于0值的都返回1**，其他的则返回原值。

由此可知，**分值越低**的则**越不会被kill**，而且该值可以通过**修改oom\_score\_adj进行调整**。

#### 11.1.1.2 oom\_kill\_process()进行kill操作

判断**当前被kill的进程情况**，如果该进程处于**退出状态**，则设置**TIF\_MEMDIE标志**，不做kill操作；

接着会通过**list\_for\_each\_entry**()遍历该**进程的子进程信息**，如果**某个子进程**拥有**不同的mm**且**合适被kill掉**，将会**优先考虑**将该**子进程替代父进程kill掉**，这样可以避免kill掉父进程带来的接管子进程的工作开销；

再往下通过find\_lock\_task\_mm()找到**持有mm锁的进程**，如果进程处于**退出状态**，则return，否则继续处理，若此时的进程与传入的不是同一个时则更新victim；

继而接着通过**for\_each\_process**()查找与**当前被kill进程**使用到了**同样的共享内存的进程**进行**一起kill掉**，**kill之前**将**对应的进程**添加**标识TIF\_MEMDIE**，而kill的动作则是通过**发送SICKILL信号给对应进程(！！！**)，由被kill进程从**内核态返回用户态时进行处理(！！！**)。

至此，OOM kill处理分析完毕。

# 12 slab slob slub分配器

明显**每次分配小于一个页面**的都统一**分配一个页面的空间**是过于**浪费**且不切实际的，因此必须**充分利用未被使用的空闲空间**，同时也要**避免过多地访问操作页面分配**。基于该问题的考虑，内核需要**一个缓冲池**对小块内存进行有效的管理起来，于是就有了slab内存分配算法。

每次**小块内存**的分配**优先来自于该内存分配器**，**小块内存的释放**也是**先缓存至该内存分配器**，留作下次申请时进行分配，避免了频繁分配和释放小块内存所带来的额外负载。而这些**被管理的小块内存**在管理算法中被视之为“**对象**”。

**提供小内存块**不是**slab分配器**的唯一任务.由于结构上的特点.**它也用作一个缓存**.主要针对**经常分配并释放的对象**. 通过**建立slab缓存**, 内核能够**储备一些对象**, 供后续使用, 即使在**初始化状态**, 也是如此.

**slab分配器**将**释放的内存块**保存在一个**内部列表**中.并**不马上返回给伙伴系统**.在**请求**为**该类对象**分配一个**新实例**时,会使用**最近释放的内存块**.这有两个优点.首先,由于内核**不必使用伙伴系统算法**,处理时间会变短.其次,由于该**内存块仍然是"新**"的，因此其仍然**驻留**在**CPU高速缓存的概率较高**.

slab分配器还有**两个**更进一步的**好处**

- **调用伙伴系统**的操作对系统的**数据和指令高速缓存**有相当的影响。slab分配器在可能的情况下**减少了对伙伴系统的调用**,有助于防止不受欢迎的缓存"污染".

- 如果**数据存储**在**伙伴系统直接提供的页**中，那么其**地址**总是出现在**2的幂次的整数倍附近**(许多**将页划分为更小块**的其他分配方法,也有同样的特征).这**对CPU高速缓存的利用有负面**影响,由于这种地址分布,使得**某些缓存行过度使用**,而**其他的则几乎为空**.多处理器系统可能会加剧这种不利情况,因为不同的内存地址可能在不同的总线上传输,上述情况会导致某些总线拥塞,而其他总线则几乎没有使用.

通过**slab着色(slab coloring**), slab分配器能够**均匀地分布对象**, 以实现**均匀的缓存利用**

**着色**这个术语是隐喻性的. 它与颜色无关,只是表示slab中的**对象需要移动的特定偏移量**,以便**使对象放置到不同的缓存行**.

**slab分配器**由何得名?各个**缓存管理的对象**，会合并为**较大的组**，覆盖**一个或多个连续页帧**.这种**组称作slab**，**每个缓存**由**几个这种slab组成**.

**Slab内存分配算法**是**最先出现**的；

后来被改进适用于嵌入式设备，以满足**内存较少的情况**下的使用，该改进后的算法**占用资源极少**，其被称之为**slob内存分配算法**；

再后来由于**slab内存分配算法**的**管理结构较大且设计复杂**，再一次被改进简化，而该简化改进的算法称之为**slub内存分配算法**。

当前**linux内核**中对**该三种算法是都提供**的，但是在**编译内核**的时候**仅可以选择其一进行编译(！！！**)，鉴于slub比slab分配算法更为简洁和便于调试，在linux 2.6.22版本中，**slub分配算法替代了slab内存管理算法**的代码。

## 12.1 操作接口

虽然该三种算法的实现存在差异，但是其**对外提供的API接口都是一样**的

**创建一个slab**的API为：

```c
struct kmem_cache *kmem_cache_create(const char *name, size_t size, size_t align,         unsigned long flags, void (*ctor)(void *))
```

其中**name**为**slab的名称**，size为**slab**中**每个对象的大小**，**align**为**内存对齐基值**，**flags**为使用的**标志(即分配掩码**)，而**ctor**为**构造函数**；

**销毁slab**的API为：

```c
void kmem_cache_destroy(struct kmem_cache *s)
```

从**分配算法**中**分配一个对象**：

```c
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
```

将**对象释放到分配算法**中：

```c
void kmem_cache_free(struct kmem_cache *cachep, void *objp)
```

除此之外，该分配算法还内建了一些slab（2\^KMALLOC\_SHIFT\_LOW \– 2\^KMALLOC\_SHIFT\_HIGH）用于**不需要特殊处理的对象分配**。

除了上面列举的分配和释放对象的接口，实际上对用户可见的接口为**kmalloc**及**kfree**。

一个**大量使用slab机制**的是**kmallc()函数**接口。**kmem\_cache\_create**()函数用于**创建自己的缓存描述符**，**kmalloc**()函数用于**创建通用的缓存**，kfree()用于释放缓存, 类似于**用户空间中C标准库malloc**()函数。

- kmalloc、\_\_kmalloc和kmalloc\_node是一般的(特定于结点)**内存分配**函数.

- kmem\_cache\_alloc、kmem\_cache\_alloc\_node提供（特定于结点）**特定类型的内核缓存**.

## 12.2 内核中的内存管理

- **kmalloc(size, flags**)分配长度为**size字节**的一个内存区, 并返回指向该内存区**起始处的一个void指针**. 如果没有足够内存(在内核中这种情形不大可能,但却始终要考虑到),则结果为NULL指针.**flags参数**使用之前讨论的**GFP\_常数**，来**指定**分配内存的**具体内存域**，例如GFP\_DMA指定分配适合于**DMA的内存区**.

- kfree(\*ptr)释放\*ptr指向的内存区.

与**用户空间程序**设计相比,内核还包括**percpu\_alloc**和**percpu\_free**函数，用于为**各个系统CPU**分配和释放所需**内存区**(不是明确地用于当前活动CPU).

用kmalloc分配的内存区,首先通过**类型转换**变为正确的类型, 然后赋值到**指针变量**.

>info = (struct cdrom\_info *) kmalloc(sizeof(struct cdrom\_info), GFP\_KERNEL);

下面看一个例子, 创建一个名为"fig\_object"的**slab描述符**, 大小为**20 Byte**, align为**8 Byte**, flags为0, 假设L1 Cache line大小为16 Byte, 那么可以编写一个简单内核模块实现上述需求.

必须首先用**kmem\_cache\_create**建立一个**适当的缓存**,接下来即可使用**kmem\_cache\_alloc**和**kmem\_cache\_free**分配和释放**其中包含的对象**。**slab分配器**负责完成与伙伴系统的交互，来**分配所需的页**.

```c
static struct kmem_cache *fcache
static void *buf;
// 举例：创建名为"figo_object"的slab描述符，大小为20Byte，8字节Byte
static int —init fcache_init(void)
{
    fcache = kmem_cache_create ("figo_object", 20, 8, 0, NULL ) ；
    if (!fcache) {
        kmem_cache_destroy (fcache);
        return -ENOMEM;
    }
    buf = kmem_cache_alloc(fcache, GFP_KERNEL);
    return 0;
}
static void _exit fcache_exit(void)
{
    kmem_cache_free(fcache, buf);
    kmem_cache_destroy(fcache);
}
module_init(fcache_init);
module_exit(fcache_exit);
```

所有**活动缓存的列表**保存**在/proc/slabinfo**中(为节省空间，下文的输出省去了不重要的部分).

![cat /proc/slabinfo](./images/29.png)

我们可以从其中检索到一些**特定的高速缓存的信息**, 比如**task\_struct, mm\_struct**等.

![cat /proc/slabinfo](./images/30.png)

输出的各列除了包含用于标识各个缓存的**字符串名称**(也**确保不会创建相同的缓存**)之外,还包含下列信息.

- 缓存中**活动对象的数量**。

- 缓存中**对象的总数（已用和未用**）。

- 所管理**对象的长度**，按**字节计算**。

- 一个slab中**对象的数量**。

- 每个slab中**页的数量**。

- **活动slab的数量**。

在内核决定**向缓存分配更多内存**时,所**分配对象的数量**.每次会分配一个**较大的内存块**,以**减少与伙伴系统的交互**.在**缩小缓存**时，也使用该值作为**释放内存块的大小**.

# 13 slab分配的原理



# 14 slub原理

## 14.1 slub数据结构

首先鸟瞰全局，由下图进行入手分析slub内存分配算法的管理框架：

![config](./images/28.png)

沿用slab算法对**对象**及**对象缓冲区**的称呼，**一个slab**表示**某个特定大小空间的缓冲片区**，而**片区里面**的**一个特定大小的空间**则称之为**对象**。

Slub内存分配算法中，**每种slab的类型**都是由**kmem\_cache**类型的数据结构来描述的。该结构的定义为：

```c
[include/linux/slub_def.h]
struct kmem_cache {
    /*每CPU的结构，用于各个CPU的缓存管理*/
	struct kmem_cache_cpu __percpu *cpu_slab;
	/* Used for retriving partial slabs etc */
	/*描述标识，描述缓冲区属性的标识*/
	unsigned long flags;
	unsigned long min_partial;
	/*对象大小，包括元数据大小*/
	int size;		/* The size of an object including meta data */
	/*slab对象纯大小*/
	int object_size;	/* The size of an object without meta data */
	/*空闲对象的指针偏移*/
	int offset;		/* Free pointer offset. */
	/*每CPU持有量*/
	int cpu_partial;	/* Number of per cpu partial objects to keep around */
	/*存放分配给slab的页框的阶数(高16位)和slab中的对象数量(低16位)*/
	struct kmem_cache_order_objects oo;

	/* Allocation and freeing of slabs */
	/*存储单对象所需的页框阶数及该阶数下可容纳的对象数*/
	struct kmem_cache_order_objects max;
	struct kmem_cache_order_objects min;
	/*申请页面时使用的GFP标识*/
	gfp_t allocflags;	/* gfp flags to use on each alloc */
	/*缓冲区计数器，当用户请求创建新的缓冲区时，SLUB分配器重用已创建的相似大小的缓冲区，从而减少缓冲区个数*/
	int refcount;		/* Refcount for slab cache destroy */
	/*创建对象的回调函数*/
	void (*ctor)(void *);
	/*元数据偏移量*/
	int inuse;		/* Offset to metadata */
	/*对齐值*/
	int align;		/* Alignment */
	int reserved;		/* Reserved bytes at the end of slabs */
	/*slab缓存名称*/
	const char *name;	/* Name (only for display!) */
	/*用于slab cache管理的链表*/
	struct list_head list;	/* List of slab caches */
	int red_left_pad;	/* Left redzone padding size */
#ifdef CONFIG_SYSFS
	struct kobject kobj;	/* For sysfs */
#endif
#ifdef CONFIG_MEMCG
	struct memcg_cache_params memcg_params;
	int max_attr_size; /* for propagation, maximum size of a stored attr */
#ifdef CONFIG_SYSFS
	struct kset *memcg_kset;
#endif
#endif

#ifdef CONFIG_NUMA
	/*
	 * Defragmentation by allocating from a remote node.
	 */
	/*远端节点的反碎片率，值越小则越倾向从本节点中分配对象*/
	int remote_node_defrag_ratio;
#endif
    /*各个内存管理节点的slub信息*/
	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

这里主要关注的是cpu\_slab和node成员。

其中cpu\_slab的结构类型是**kmem\_cache\_cpu**，是**Per\-CPU类型**数据，**各个CPU**都有**自己独立的一个结构**，用于**管理本地的对象缓存**。具体的结构定义如下：

```c
[include/linux/slub_def.h]
struct kmem_cache_cpu {
    /* 空闲对象队列的指针 */
    void **freelist; /* Pointer to next available object */ 
    /* 用于保证cmpxchg_double计算发生在正确的CPU上，并且可作为一个锁保证不会同时申请这个kmem_cache_cpu的对象 */
    unsigned long tid; /* Globally unique transaction id */ 
    /* 指向slab对象来源的内存页面 */
    struct page *page; /* The slab from which we are allocating */ 
    /* 指向曾分配完所有的对象，但当前已回收至少一个对象的page */
    struct page *partial; /* Partially allocated frozen slabs */ 
#ifdef CONFIG_SLUB_STATS
    unsigned stat[NR_SLUB_STAT_ITEMS];
#endif
};
```

kmem\_cache\_node结构：

```c
[/mm/slab.h]
struct kmem_cache_node {
    /*保护结构内数据的自旋锁*/
    spinlock_t list_lock; 
 
#ifdef CONFIG_SLAB
    struct list_head slabs_partial; /* partial list first, better asm code */
    struct list_head slabs_full;
    struct list_head slabs_free;
    unsigned long free_objects;
    unsigned int free_limit;
    unsigned int colour_next; /* Per-node cache coloring */
    struct array_cache *shared; /* shared per node */
    struct array_cache **alien; /* on other nodes */
    unsigned long next_reap; /* updated without locking */
    int free_touched; /* updated without locking */
#endif
 
#ifdef CONFIG_SLUB
    /*本节点的Partial slab的数目*/
    unsigned long nr_partial; 
    /*Partial slab的双向循环队列*/
    struct list_head partial; 
#ifdef CONFIG_SLUB_DEBUG
    atomic_long_t nr_slabs;
    atomic_long_t total_objects;
    struct list_head full;
#endif
#endif
 
};
```

该结构是**每个node节点**都会有的一个结构，主要是用于**管理node节点**的**slab缓存区**。

Slub分配管理中，**每个CPU都有自己的缓存管理**，也就是**kmem\_cache\_cpu数据结构管理**；而**每个node节点**也有**自己的缓存管理**，也就是**kmem\_cache\_node数据结构**管理。

**对象分配**时：

1、**当前CPU缓存**有**满足申请要求的对象**时，将会首先从**kmem\_cache\_cpu的空闲链表freelist**将对象分配出去；

2、如果**对象不够**时，将会向**伙伴管理算法中申请内存页面**，申请来的页面将会**先填充到node节点**中，然后从**node节点**取出对象到**CPU的缓存空闲链表**中；

3、如果**原来申请的node节点A**的对象，现在改为**申请node节点B**的，那么将会把**node节点A的对象释放后再申请**。

**对象释放**时：

1、会先**将对象释放到CPU上**面，如果**释放的对象**恰好与**CPU的缓存**来自**相同的页面**，则**直接添加到列表**中；

2、如果释放的对象**不是当前CPU缓存的页面**，则会把**当前的CPU的缓存对象**放到**node上**面，然后再**把该对象释放到本地的cache**中。

为了避免**过多的空闲对象缓存在管理框架**中，slub设置了**阀值**，如果**空闲对象个数**达到了**一个峰值**，将会**把当前缓存释放到node节点**中，当node节点也过了阀值，将会把node节点的对象释放到伙伴管理算法中。

## 14.2 slub初始化

在调用mem\_init()初始化伙伴管理算法后，紧接着调用的kmem\_cache\_init()便是slub分配算法的入口。其中该函数在/mm目录下有三处实现slab.c、slob.c和slub.c，表示不同算法下其初始化各异，分析slub分配算法则主要分析slub.c的实现。

```c
void __init kmem_cache_init(void)
{
	static __initdata struct kmem_cache boot_kmem_cache,
		boot_kmem_cache_node;

	if (debug_guardpage_minorder())
		slub_max_order = 0;

	kmem_cache_node = &boot_kmem_cache_node;
	kmem_cache = &boot_kmem_cache;

	create_boot_cache(kmem_cache_node, "kmem_cache_node",
		sizeof(struct kmem_cache_node), SLAB_HWCACHE_ALIGN);

	register_hotmemory_notifier(&slab_memory_callback_nb);

	/* Able to allocate the per node structures */
	slab_state = PARTIAL;

	create_boot_cache(kmem_cache, "kmem_cache",
			offsetof(struct kmem_cache, node) +
				nr_node_ids * sizeof(struct kmem_cache_node *),
		       SLAB_HWCACHE_ALIGN);

	kmem_cache = bootstrap(&boot_kmem_cache);

	kmem_cache_node = bootstrap(&boot_kmem_cache_node);

	/* Now we can use the kmem_cache to allocate kmalloc slabs */
	setup_kmalloc_cache_index_table();
	create_kmalloc_caches(0);

#ifdef CONFIG_SMP
	register_cpu_notifier(&slab_notifier);
#endif

	pr_info("SLUB: HWalign=%d, Order=%d-%d, MinObjects=%d, CPUs=%d, Nodes=%d\n",
		cache_line_size(),
		slub_min_order, slub_max_order, slub_min_objects,
		nr_cpu_ids, nr_node_ids);
}
```

register\_hotmemory\_notifier()和register\_cpu\_notifier()主要是用于注册内核通知链回调的；

create\_boot\_cache()用于创建分配算法缓存，主要是把**boot\_kmem\_cache\_node结构初始化**了。其内部的calculate\_alignment()主要用于**计算内存对齐值**，而\_\_**kmem\_cache\_create**()则是创建缓存的核心函数，其主要是把**kmem\_cache结构初始化**了。具体的\_\_kmem\_cache\_create()实现将在后面的slab创建部分进行详细分析。

至此，create\_boot\_cache()函数创建**kmem\_cache\_node对象缓冲区**完毕，往下**register\_hotmemory\_notifier**()注册**内核通知链回调**之后，同样是通过create\_boot\_cache()创建kmem\_cache对象缓冲区。接续往下走，可以看到bootstrap()函数调用，bootstrap(&boot\_kmem\_cache)及bootstrap(&boot\_kmem\_cache\_node)。

bootstrap()函数主要是将临时kmem\_cache向最终kmem\_cache迁移，并修正相关指针，使其指向最终的kmem\_cache。

# 15 kmalloc和kfree实现

## 15.1 kmalloc的实现

kmalloc()是基于slab/slob/slub分配分配算法上实现的，不少地方将其作为slab/slob/slub分配算法的入口，实际上是略有区别的。

```c
static __always_inline void *kmalloc(size_t size, gfp_t flags)
{
    if (__builtin_constant_p(size)) {
        if (size > KMALLOC_MAX_CACHE_SIZE)
            return kmalloc_large(size, flags);
#ifndef CONFIG_SLOB
        if (!(flags & GFP_DMA)) {
            int index = kmalloc_index(size);
 
            if (!index)
                return ZERO_SIZE_PTR;
 
            return kmem_cache_alloc_trace(kmalloc_caches[index],
                    flags, size);
        }
#endif
    }
    return __kmalloc(size, flags);
}
```

kmalloc()的参数**size**表示**申请的空间大小**，而**flags**则表示**分配标志**。kamlloc的分配标志众多，各标志都分配标识特定的bit位，藉此可以多样组合。

GFP\_USER：用于表示为用户空间分配内存，可能会引起休眠；

GFP\_KERNEL：内核内存的常规分配，可能会引起休眠；

GFP\_ATOMIC：该分配不会引起休眠，但可能会使用应急内存资源，通常用于中断处理中；

GFP\_HIGHUSER：使用高端内存进行分配；

GFP\_NOIO：分配内存时，禁止任何IO操作；

GFP\_NOFS：分配内存时，禁止任何文件系统操作；

GFP\_NOWAIT：分配内存时禁止休眠；

\_\_GFP\_THISNODE：分配内存时，仅从本地节点内存中分配；

GFP\_DMA：从DMA内存中分配合适的内存，应仅使用于kmalloc的cache分配；

\_\_GFP\_COLD：用于请求分配冷热页中的冷页；

\_\_GFP\_HIGH：用于表示该分配优先级较高并可能会使用应急内存资源；

\_\_GFP\_NOFAIL：用于指示该分配不允许分配失败，该标志需要慎用；

\_\_GFP\_NORETRY：如果分配内存未能够直接获取到，则不再尝试分配，直接放弃；

\_\_GFP\_NOWARN：如果分配过程中失败，不上报任何告警；

\_\_GFP\_REPEAT：如果分配过程中失败，则尝试再次申请；

函数入口if判断内的\_\_builtin\_constant\_p是**Gcc内建函数**，用于判断一个值是否为**编译时常量**，是则返回true，否则返回false。也就意味着如果调用kmalloc()传入**常量**且该值**大于KMALLOC\_MAX\_CACHE\_SIZE**（即申请空间超过kmalloc()所能分配**最大cache的大小**），那么将会通过**kmalloc\_large**()进行分配；否则都将通过\_\_**kmalloc**()进行分配。

如果通过**kmalloc\_large**()进行内存分配，将会经kmalloc\_large()\-\>kmalloc\_order()\-\>\_\_get\_free\_pages()，最终通过**Buddy伙伴算法**申请所需内存。

看\_\_kmalloc()的实现

```c
[mm/slub.c]
void *__kmalloc(size_t size, gfp_t flags)
{
    struct kmem_cache *s;
    void *ret;
 
    if (unlikely(size > KMALLOC_MAX_CACHE_SIZE))
        return kmalloc_large(size, flags);
 
    s = kmalloc_slab(size, flags);
 
    if (unlikely(ZERO_OR_NULL_PTR(s)))
        return s;
 
    ret = slab_alloc(s, flags, _RET_IP_);
 
    trace_kmalloc(_RET_IP_, ret, size, s->size, flags);
 
    return ret;
}
```

判断申请**是否超过最大cache大小**，如果是则通过**kmalloc\_large**()进行分配；接着通过**申请大小**及**申请标志**调用**kmalloc\_slab**()查找**适用的kmem\_cache**；最后通过**slab\_alloc**()进行**slab分配**。

看一下kmalloc\_slab()的实现：

```c
[mm/slab_commmon.c]
/*
 * Find the kmem_cache structure that serves a given size of
 * allocation
 */
struct kmem_cache *kmalloc_slab(size_t size, gfp_t flags)
{
    int index;
 
    if (unlikely(size > KMALLOC_MAX_SIZE)) {
        WARN_ON_ONCE(!(flags & __GFP_NOWARN));
        return NULL;
    }
 
    if (size <= 192) {
        if (!size)
            return ZERO_SIZE_PTR;
 
        index = size_index[size_index_elem(size)];
    } else
        index = fls(size - 1);
 
#ifdef CONFIG_ZONE_DMA
    if (unlikely((flags & GFP_DMA)))
        return kmalloc_dma_caches[index];
 
#endif
    return kmalloc_caches[index];
}
```

申请的大小超过KMALLOC\_MAX\_SIZE最大值，则返回NULL表示失败；如果**申请大小小于192,且不为0**，将通过size\_index\_elem宏转换为下标后，经size\_index全局数组取得索引值，否则将直接通过fls()取得索引值；最后如果开启了DMA内存配置且设置了GFP\_DMA标志，将结合索引值通过kmalloc\_dma\_caches返回kmem\_cache管理结构信息，否则将通过kmalloc\_caches返回该结构。

由此可以看出kmalloc()实现较为简单，其分配所得的**内存**不仅是**虚拟地址上的连续**存储空间，同时也是**物理地址上的连续**存储空间。这是有别于后面将会分析到的vmalloc()申请所得的内存。

## 15.2 kfree()的实现

主要是在slab.c/slob.c/slub.c中

```c
[mm/slub.c]
void kfree(const void *x)
{
    struct page *page;
    void *object = (void *)x;
 
    trace_kfree(_RET_IP_, x);
 
    if (unlikely(ZERO_OR_NULL_PTR(x)))
        return;
 
    page = virt_to_head_page(x);
    if (unlikely(!PageSlab(page))) {
        BUG_ON(!PageCompound(page));
        kfree_hook(x);
        __free_memcg_kmem_pages(page, compound_order(page));
        return;
    }
    slab_free(page->slab_cache, page, object, _RET_IP_);
}
```

经过trace\_kfree()记录kfree轨迹，然后if (unlikely(ZERO\_OR\_NULL\_PTR(x)))对**地址做非零判断**，接着virt\_to\_head\_page(x)将**虚拟地址转换到页面**；再是判断if (unlikely(!PageSlab(page)))**判断该页面是否作为slab分配管理**，如果是的话则转为通过**slab\_free()进行释放**，否则将进入if分支中；在if分支中，将会kfree\_hook()做释放前kmemleak处理（该函数主要是封装了kmemleak\_free()），完了之后将会\_\_free\_memcg\_kmem\_pages()将页面释放，同时该函数内也将cgroup释放处理。

# 16 内存破坏检测kmemcheck分析

# 17 内存泄漏检测kmemleak分析

memleak的工作原理很简单，主要是对kmalloc()、vmalloc()、kmem\_cache\_alloc()等接口分配的内存地址空间进行跟踪，通过对其地址、空间大小、分配调用栈等信息添加到PRIO搜索树中进行管理。当有匹配的内存释放操作时，将会把跟踪的信息从kmemleak管理中移除。

# 18 vmalloc不连续内存管理

**伙伴管理算法**初衷是**解决外部碎片**问题，而**slab算法**则是用于**解决内部碎片**问题，但是内存使用的得不合理终究会产生碎片。碎片问题产生后，申请大块连续内存将可能持续失败，但是实际上内存的空闲空间却是足够的。这时候就引入了**不连续页面管理算法**，即我们常用的**vmalloc**申请分配的内存空间，它主要是用于将**不连续的页面**，通过内存映射到**连续的虚拟地址空间**中，提供给申请者使用，由此实现内存的高利用。

# 19 缺页异常处理

