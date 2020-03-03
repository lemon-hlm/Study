[TOC]

- 1 前景回顾
    - 1.1 Linux内存管理的层次结构
    - 1.2 start\_kernel启动过程
    - 1.3 setup\_arch设置
- 2 引导内存分配器bootmem概述  
    - 2.1 初始化阶段的引导内存分配器bootmem
    - 2.2 为什么需要bootmem
    - 2.3 为什么在系统运行时抛弃bootmem    
- 3 引导内存分配器数据结构
    - 3.1 bootmem\_data描述内存引导区
- 4 初始化引导分配器
    - 4.1 IA-32的初始化
- 5 bootmem分配内存接口        
    - 5.1 UMA结构的分配函数
        - 5.1.1 从ZONE\_NORMAL区域分配函数
        - 5.1.2 从ZONE\_DMA区域分配函数
        - 5.1.3 函数实现方式
    - 5.2 NUMA结构下的分配函数
        - 5.2.1 分配函数接口
        - 5.2.2 函数实现方式
    - 5.3 \_\_alloc\_memory\_core进行内存分配
- 6 bootmem释放内存
- 7 停用bootmem
- 8 链接

当buddy系统和slab分配器初始化好后，在**mem\_init**()中对bootmem分配器进行释放，内存管理与分配由buddy系统，slab分配器等进行接管。

bootmem分配器使用一个**bitmap**来**标记物理页**是否被占用，分配的时候按照**第一适应**的原则，从bitmap中进行查找，如果这位为1，表示已经被占用，否则表示未被占用。为什么系统运行的时候不使用bootmem分配器呢？

- bootmem分配器**每次在bitmap中进行线性搜索**，效率非常低，
- 在内存的**起始端留下许多小的空闲碎片**，在需要**非常大的内存块**的时候，检查位图这一过程就显得代价很高。

bootmem分配器是用于在启动阶段分配内存的，对该分配器的需求集中于简单性方面，而不是性能和通用性。

# 1 前景回顾

## 1.1 Linux内存管理的层次结构

Linux把物理内存划分为三个层次来管理

| 层次 | 描述 |
|:----|:----|
| 存储节点(Node) |  CPU被划分为多个节点(node), 内存则被分簇, 每个CPU对应一个本地物理内存, 即一个CPU-node对应一个内存簇bank，即每个内存簇被认为是一个节点 |
| 管理区(Zone)   | 每个物理内存节点node被划分为多个内存管理区域, 用于表示不同范围的内存, 内核可以使用不同的映射方式映射物理内存 |
| 页面(Page) 	   |	内存被细分为多个页面帧, 页面是最基本的页面分配的单位　｜

为了支持NUMA模型，也即CPU对不同内存单元的访问时间可能不同，此时系统的物理内存被划分为几个节点(node), 一个node对应一个内存簇bank，即每个内存簇被认为是一个节点

- 首先, 内存被划分为**结点**. 每个节点关联到系统中的一个处理器, 内核中表示为`pg_data_t`的实例. 系统中每个节点被链接到一个以NULL结尾的`pgdat_list`链表中<而其中的每个节点利用`pg_data_tnode_next`字段链接到下一节．而对于PC这种UMA结构的机器来说, 只使用了一个成为contig\_page\_data的静态pg\_data\_t结构.

- 接着各个节点又被划分为内存管理区域, 一个**管理区域**通过struct zone\_struct描述, 其被定义为zone\_t,用以表示内存的某个范围,低端范围的16MB被描述为ZONE\_DMA,某些工业标准体系结构中的(ISA)设备需要用到它,然后是可直接映射到内核的普通内存域ZONE\_NORMAL,最后是超出了内核段的物理地址域ZONE\_HIGHMEM, 被称为高端内存.　是系统中预留的可用内存空间, 不能被内核直接映射.

- 最后**页帧(page frame)**代表了系统内存的最小单位, 堆内存中的每个页都会创建一个struct page的一个实例. 传统上，把内存视为连续的字节，即内存为字节数组，内存单元的编号(地址)可作为字节数组的索引. 分页管理时，将若干字节视为一页，比如4K byte. 此时，内存变成了连续的页，即内存为页数组，每一页物理内存叫页帧，以页为单位对内存进行编号，该编号可作为页数组的索引，又称为页帧号.

## 1.2 start\_kernel启动过程

首先我们来看看start\_kernel是如何初始化系统的, start\_kernel定义在[init/main.c?v=4.7, line 479](http://lxr.free-electrons.com/source/init/main.c?v=4.7#L479)

其代码很复杂, 我们只截取出其中与内存管理初始化相关的部分, 如下所示

```cpp
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
| [setup\_arch](http://lxr.free-electrons.com/ident?v=4.7;i=setup_arch) | 是一个特定于体系结构的设置函数, 其中一项任务是负责**初始化自举分配器** |
| [mm\_init\_cpumask](http://lxr.free-electrons.com/source/include/linux/mm_types.h?v=4.7#L522) | 初始化**CPU屏蔽字** |
| [setup\_per\_cpu\_areas](http://lxr.free-electrons.com/ident?v=4.7;i=setup_per_cpu_areas) | 函数[(查看定义)](http://lxr.free-electrons.com/source/mm/percpu.c?v4.7#L2205])给每个CPU分配内存，并拷贝.data.percpu段的数据.为系统中的**每个CPU的per\_cpu变量申请空**间.<br>在SMP系统中,setup\_per\_cpu\_areas初始化源代码中(使用[per_cpu宏](http://lxr.free-electrons.com/source/include/linux/percpu-defs.h#L256))定义的静态per-cpu变量,这种变量对系统中每个CPU都有一个独立的副本. <br>此类变量保存在内核二进制影像的一个独立的段中,setup\_per\_cpu\_areas的目的就是为系统中各个CPU分别创建一份这些数据的副本<br>在**非SMP系统**中这是一个**空操作** |
| [build\_all\_zonelists](http://lxr.free-electrons.com/source/mm/page_alloc.c?v4.7#L5029) | 建立并初始化**结点**和**内存域**的数据结构 |
| [mm\_init](http://lxr.free-electrons.com/source/init/main.c?v4.7#L464) | 建立了内核的**内存分配器**,<br>其中通过[**mem\_init**](http://lxr.free-electrons.com/ident?v=4.7&i=mem_init)**停用bootmem**分配器并迁移到实际的内存管理器(比如伙伴系统)<br>然后调用**kmem\_cache\_init**函数初始化内核内部**用于小块内存区的分配器** |
| [kmem\_cache\_init\_late](http://lxr.free-electrons.com/source/mm/slab.c?v4.7#L1378) | 在**kmem\_cache\_init**之后,完善分配器的**缓存机制**,　当前3个可用的内核内存分配器[slab](http://lxr.free-electrons.com/source/mm/slab.c?v4.7#L1378), [slob](http://lxr.free-electrons.com/source/mm/slob.c?v4.7#L655), [slub](http://lxr.free-electrons.com/source/mm/slub.c?v=4.7#L3960)都会定义此函数　|
| [kmemleak\_init](http://lxr.free-electrons.com/source/mm/kmemleak.c?v=4.7#L1857) | Kmemleak工作于内核态，Kmemleak提供了一种**可选的内核泄漏检测**，其方法类似于**跟踪内存收集器**。当独立的对象没有被释放时，其报告记录在 [/sys/kernel/debug/kmemleak](http://lxr.free-electrons.com/source/mm/kmemleak.c?v=4.7#L1467)中, Kmemcheck能够帮助定位大多数内存错误的上下文 |
| [setup\_per\_cpu\_pageset](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L5392) | **初始化CPU高速缓存行**, 为pagesets的第一个数组元素分配内存, 换句话说, 其实就是第一个系统处理器分配<br>由于在分页情况下，**每次存储器访问都要存取多级页表**，这就大大降低了访问速度。所以，为了提高速度，在CPU中设置一个最近存取页面的**高速缓存硬件机制**，当进行存储器访问时，先检查要访问的页面是否在高速缓存中. |

## 1.3 setup_arch设置

# 2 引导内存分配器bootmem概述

由于硬件配置多种多样, 所以在编译时就静态初始化所有的内核存储结构是不现实的.

bootmem分配器是系统启动初期的内存分配方式，在耳熟能详的伙伴系统建立前内存都是利用bootmem分配器来分配的，伙伴系统框架建立起来后，bootmem会过渡到伙伴系统.

## 2.1 初始化阶段的引导内存分配器bootmem

**引导内存分配器(boot memory allocator--bootmem分配器)**基于最先适配(first-first)分配器的原理(这儿是很多系统的内存分配所使用的原理), 使用**一个位图**来管理**页**,以位图代替原来的空闲链表结构来表示存储空间,**位图的比特位的数目**与系统中**物理内存页面数目相同**.若位图中某一位是1,则标识该页面已经被分配(已用页),否则表示未被占有(未用页).

在需要分配内存时, 分配器逐位的**扫描位图**,直至找到一个能提供**足够连续页的位置**,即所谓的最先最佳(first-best)或最先适配位置.

该分配机制通过记录上一次**分配的页面帧号(PFN)结束时的偏移量**来实现分配大小小于一页的空间, 连续的小的空闲空间将被合并存储在一页上.

## 2.2 为什么需要bootmem

## 2.3 为什么在系统运行时抛弃bootmem

当系统运行时, 为何不继续使用bootmem分配机制呢?

- 其中一个关键原因在于 : 但它每次分配都必须**从头扫描位图**,每次通过对内存域进行**线性搜索**来实现分配.

- 其次**首先适应算法**容易在内存的**起始断留下许多小的空闲碎片**,在需要分配**较大的空间页**时,检查位图的成本将是非常高的.

引导内存分配器bootmem分配器简单却非常低效,　因此在内核完全初始化之后,　不能将该分配器继续内存管理,而伙伴系统(连同slab, slub或者slob分配器)是一个好很多的备选方案．

# 3 引导内存分配器数据结构

内核用**bootmem\_data**表示**引导内存区域**

即使是初始化用的最先适配分配器也必须使用一些数据结构存, 内核为系统中每一个结点都提供了一个**struct bootmem\_data结构**的实例, 用于bootmem的内存管理. 它含有引导内存分配器给结点分配内存时所需的信息. 当然, 这时候**内存管理还没有初始化**,因而**该结构所需的内存是无法动态分配**的,必须在**编译时分配给内核**.

在UMA系统上该分配的实现与CPU无关,而**NUMA系统**内存结点与CPU相关联,因此采用了**特定体系结构的解决方法**.

## 3.1 bootmem\_data描述内存引导区

bootmem\_data的结构定义在[include/linux/bootmem.h?v=4.7, line 28](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L28), 其定义如下所示

```cpp
#ifndef CONFIG_NO_BOOTMEM
/*
* node_bootmem_map is a map pointer - the bits represent all physical 
* memory pages (including holes) on the node.
*/
typedef struct bootmem_data {
       unsigned long node_min_pfn;
       unsigned long node_low_pfn;
       void *node_bootmem_map;
       unsigned long last_end_off;
       unsigned long hint_idx;
       struct list_head list;
} bootmem_data_t;

extern bootmem_data_t bootmem_node_data[];

#endif
```

|  字段  |  描述  |
|:-----|:------|
| node\_min\_pfn | **节点起始地址** |
| node\_low_pfn | **低端内存最后一个page的页帧号** |
| node\_bootmem\_map | 指向内存中**位图bitmap**所在的位置 |
| last\_end\_off | 分配的**最后一个页内的偏移**，如果**该页完全使用，则offset为0** |
| hint\_idx | |
| list | |

**bootmem的位图**建立在从**start\_pfn**开始的地方, 也就是说, **内核映像终点\_end上方的地方**. 这个位图用来**管理低区**（例如小于 896MB), 因为在0到896MB的范围内, 有些页面可能保留, 有些页面可能有空洞, 因此, 建立这个位图的目的就是要搞清楚**哪一些物理页面是可以动态分配**的

- node\_bootmem\_map就是一个指向位图的指针.node\_min\_pfn表示存放bootmem位图的**第一个页面**(即**内核映像结束处的第一个页面**)

- node\_low\_pfn 表示**物理内存的顶点**, 最高不超过896MB

# 4 初始化引导分配器

系统是从**start\_kernel**开始启动的, 在启动过程中通过调用**体系结构相关**的**setup\_arch**函数, 来获取初始化**引导内存分配器所需的参数信息**,各种**体系结构**都有**对应的函数来获取**这些信息,在获取信息完成后, 内核首先初始化了bootmem自身, 然后接着又用bootmem分配和初始化了内存结点和管理域, 

因此初始化bootmem的工作主要分成两步

- 初始化bootmem自身的**数据结构**

- 用bootmem初始化**内存结点管理域**

bootmem分配器的初始化是一个特定于体系结构的过程, 此外还取决于系统的内存布局

## 4.1 IA-32的初始化

在使用bootmem, 内核在**setup\_arch函数**中通过[**setup\_memory**](http://lxr.free-electrons.com/source/arch/i386/kernel/setup.c?v=2.4.37#L1045)来分析检测到的内存区,以找到**低端内存区中最大的页帧号**。由于**高端内存处理太麻烦**，由此**对bootmem分配器无用**。**全局变量max\_low\_pfn**保存了可映射的**最高页的编号**。内核会在启动日志中报告找到的内存的数量。

i386上的初始化过程:

![i386上的初始化过程](./images/i386-setup_memory.jpg)

# 5 bootmem分配内存接口

bootmem提供了各种函数用于在**初始化期间分配内存**.

尽管[mm/bootmem.c](http://lxr.free-electrons.com/source/mm/nobootmem.c?v4.7)中提供了一些了的内存分配函数，但是这些函数大多数以\__下划线开头, 这个标识告诉我们尽量不要使用他们, 他们过于底层, 往往是不安全的, 因此特定于某个体系架构的代码并没有直接调用它们，而是通过[linux/bootmem.h](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v4.7)提供的一系列的宏

## 5.1 UMA结构的分配函数

首先我们讲解一下子在**UMA系统**中, 可供使用的函数

## 5.1.1 从ZONE\_NORMAL区域分配函数

下面列出的这些函数可以从ZONE_NORMAL内存域分配指向大小的内存

| 函数 | 描述 | 定义 |
|:---:|:----|:---:|
|alloc\_bootmem(size) | 按照指定大小在**ZONE\_NORMAL内存域**分配函数. 数据是对齐的, 这使得内存或者从可适用于**L1高速缓存**的理想位置开始| [alloc\_bootmem](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L122)<br>[\_\_alloc\_bootmem](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L700)<br>[\_\_\_alloc\_bootmem](http://lxr.free-electrons.com/source/mm/bootmem.c?=4.7#L672) |
|alloc\_bootmem\_align(x, align) | 同alloc\_bootmem函数, 按照指定大小在ZONE\_NORMAL内存域分配函数, 并按照**align进行数据对齐** | [alloc\_bootmem\_align](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L124)<br>基于\_\_alloc_bootmem实现 |
|alloc\_bootmem\_pages(size)) | 同alloc\_bootmem函数, 按照指定大小在ZONE\_NORMAL内存域分配函数, 其中\_page只是指定数据的对其方式从页边界(\_\_pages)开始  |  [alloc\_bootmem\_pages](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L128)<br>基于\_\_alloc\_bootmem实现 |
|alloc\_bootmem\_nopanic(size) | alloc\_bootmem\_nopanic是最基础的通用的，一个用来尽力而为分配内存的函数，它通过list\_for\_each\_entry在**全局链表bdata\_list**中分配内存. alloc\_bootmem和alloc\_bootmem\_nopanic类似，它的底层实现首先通过alloc\_bootmem\_nopanic函数分配内存，但是一旦内存分配失败，系统将通过panic("Out of memory")抛出信息，并停止运行 | [alloc\_bootmem\_nopanic](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L126)<br>[\_\_alloc\_bootmem\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L664)<br>[\_\_\_alloc\_bootmem\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L632) |

这些函数的定义在[include/linux/bootmem.h](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L122)

```cpp
//http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L122
#define alloc_bootmem(x) \
    __alloc_bootmem(x, SMP_CACHE_BYTES, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_align(x, align) \
    __alloc_bootmem(x, align, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_nopanic(x) \
    __alloc_bootmem_nopanic(x, SMP_CACHE_BYTES, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_pages(x) \
    __alloc_bootmem(x, PAGE_SIZE, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_pages_nopanic(x) \
    __alloc_bootmem_nopanic(x, PAGE_SIZE, BOOTMEM_LOW_LIMIT)
```

## 5.1.2 从ZONE_DMA区域分配函数

下面的函数可以从ZONE\_DMA中分配内存

| 函数 | 描述 | 定义 |
|:---:|:----|:---:|
| alloc\_bootmem\_low(size) | 按照指定大小在ZONE\_DMA内存域分配函数. 类似于alloc\_bootmem, 数据是对齐的 | [alloc\_bootmem\_low\_pages\_nopanic](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L141)<br>底层基于\_\_\_alloc\_bootmem |
| alloc\_bootmem\_low\_pages\_nopanic(size) | 按照指定大小在ZONE\_DMA内存域分配函数. 类似于alloc\_bootmem\_pages, 数据在页边界对齐, 并且错误后不输出panic | [alloc\_bootmem\_low\_pages\_nopanic](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L143)<br>底层基于[\_\_alloc\_bootmem\_low\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L838)
| alloc\_bootmem\_low\_pages(size) | 按照指定大小在ZONE\_DMA内存域分配函数. 类似于alloc\_bootmem\_pages, 数据在页边界对齐 | [alloc\_bootmem\_low\_pages](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L832)<br>底层基于[\_\_alloc\_bootmem\_low\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L838) |

这些函数的定义在[include/linux/bootmem.h](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L141)

```cpp
#define alloc_bootmem_low(x) \
    __alloc_bootmem_low(x, SMP_CACHE_BYTES, 0)
#define alloc_bootmem_low_pages_nopanic(x) \
    __alloc_bootmem_low_nopanic(x, PAGE_SIZE, 0)
#define alloc_bootmem_low_pages(x) \
    __alloc_bootmem_low(x, PAGE_SIZE, 0)
```

## 5.1.3 函数实现方式

通过分析我们可以看到alloc\_bootmem\_nopanic的底层实现函数[\_\_\_alloc\_bootmem\_nopanic](http://lxr.free-electrons.com/source/mm/nobootmem.c?v4.7#L273)实现了一套最基础的内存分配函数, 而[\_\_\_alloc\_bootmem函数](http://lxr.free-electrons.com/source/mm/nobootmem.c?v=4.7#L281)则通过\_\_\_alloc\_bootmem\_nopanic函数实现, 它首先通过\_\_\_alloc\_bootmem\_nopanic函数分配内存，但是一旦内存分配失败，系统将通过`panic("Out of memory")`抛出信息，并停止运行, 其他的内存分配函数除了都是基于alloc\_bootmem\_nopanic族的函数, 都是基于_\_\_alloc\_bootmem的. 那么所有的函数都是间接的基于\_\_\_alloc\_bootmem\_nopanic实现的

```cpp
static void * __init ___alloc_bootmem(unsigned long size, unsigned long align,
                    unsigned long goal, unsigned long limit)
{
    void *mem = ___alloc_bootmem_nopanic(size, align, goal, limit);

    if (mem)
        return mem;
    /*
     * Whoops, we cannot satisfy the allocation request.
     */
    pr_alert("bootmem alloc of %lu bytes failed!\n", size);
    panic("Out of memory");
    return NULL;
}
```

所有这些分配函数最后都是间接的通过**\_\_\_alloc\_bootmem\_nopanic**函数来完成内存分配的, 该函数定义在[mm/bootmem.c?v=4.7, line 632](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L632)

```cpp
static void * __init ___alloc_bootmem_nopanic(unsigned long size,
                          unsigned long align,
                          unsigned long goal,
                          unsigned long limit)
{
    void *ptr;

restart:
    ptr = alloc_bootmem_core(size, align, goal, limit);
    if (ptr)
        return ptr;
    if (goal) {
        goal = 0;
        goto restart;
    }

    return NULL;
}
```

## 5.2 NUMA结构下的分配函数

## 5.2.1 分配函数接口

在NUMA系统上, 基本的API是相同的, 但是函数增加了\_node后缀, 与UMA系统的函数相比, 还需要一些额外的参数, 用于指定内存分配的结点.


| 函数 | 定义 |
|:---:|:---:|
|alloc\_bootmem\_node(pgdat, size) |  [alloc\_bootmem\_node](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L132)<br>[\_\_alloc\_bootmem\_node](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L777)<br>[ \_\_\_alloc\_bootmem\_node](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L708)|
|alloc\_bootmem\_node\_nopanic(pgdat, size) |  [alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L134)<br>[\_\_alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L738)<br>[\_\_\_alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L708) |
|alloc\_bootmem\_pages\_node(pgdat, size) |  [alloc\_bootmem\_pages\_node](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L136)<br>[\_\_alloc\_bootmem\_node](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L747)<br>[\_\_\_alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L708) |
|alloc\_bootmem\_pages\_node\_nopanic(pgdat, size) |  [alloc\_bootmem\_pages\_node\_nopanic](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L138)<br>[\_\_alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L738)<br>[\_\_\_alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L708) |
|alloc\_bootmem\_low\_pages\_node(pgdat, size) | [alloc\_bootmem\_low\_pages\_node](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L14)<br>[\_\_alloc\_bootmem\_low\_node](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L861)<be>[\_\_\_alloc\_bootmem\_node](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L747)<br>[\_\_\_alloc\_bootmem\_node\_nopanic](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L707) |

这些函数定义在[include/linux/bootmem.h]( http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L141)和[mm/bootmem.c](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7)

```cpp
//  http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L132
#define alloc_bootmem_node(pgdat, x) \
    __alloc_bootmem_node(pgdat, x, SMP_CACHE_BYTES, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_node_nopanic(pgdat, x) \
    __alloc_bootmem_node_nopanic(pgdat, x, SMP_CACHE_BYTES, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_pages_node(pgdat, x) \
    __alloc_bootmem_node(pgdat, x, PAGE_SIZE, BOOTMEM_LOW_LIMIT)
#define alloc_bootmem_pages_node_nopanic(pgdat, x) \
    __alloc_bootmem_node_nopanic(pgdat, x, PAGE_SIZE, BOOTMEM_LOW_LIMIT)

//  http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L147
    __alloc_bootmem_low(x, PAGE_SIZE, 0)
#define alloc_bootmem_low_pages_node(pgdat, x) \
    __alloc_bootmem_low_node(pgdat, x, PAGE_SIZE, 0)
```

## 5.2.2 函数实现方式

NUMA结构下这些分配接口的实现方式与UMA结构下类似,这些都是\_\_\_alloc\_bootmem\_node或者\_\_\_alloc\_bootmem\_node\_nopanic, 而\_\_\_alloc\_bootmem\_node函数则先通过\_\_\_alloc\_bootmem\_node\_nopanic分配内存空间, 如果出错则打印`panic("Out of memory")`, 并终止

```cpp
void * __init ___alloc_bootmem_node(pg_data_t *pgdat, unsigned long size,
                    unsigned long align, unsigned long goal,
                    unsigned long limit)
{
    void *ptr;

    ptr = ___alloc_bootmem_node_nopanic(pgdat, size, align, goal, 0);
    if (ptr)
        return ptr;

    pr_alert("bootmem alloc of %lu bytes failed!\n", size);
    panic("Out of memory");
    return NULL;
}
```

那么我们现在就进入分配函数的核心\_\_\_alloc\_bootmem\_node\_nopanic, 它定义在[mm/nobootmem.c?v=4.7, line 317](http://lxr.free-electrons.com/source/mm/nobootmem.c?v=4.7#L317)

```cpp
void * __init ___alloc_bootmem_node_nopanic(pg_data_t *pgdat,
                unsigned long size, unsigned long align,
                unsigned long goal, unsigned long limit)
{
    void *ptr;

    if (WARN_ON_ONCE(slab_is_available()))
        return kzalloc(size, GFP_NOWAIT);
again:

    /* do not panic in alloc_bootmem_bdata() */
    if (limit && goal + size > limit)
        limit = 0;

    ptr = alloc_bootmem_bdata(pgdat->bdata, size, align, goal, limit);
    if (ptr)
        return ptr;

    ptr = alloc_bootmem_core(size, align, goal, limit);
    if (ptr)
        return ptr;

    if (goal) {
        goal = 0;
        goto again;
    }

    return NULL;
}
```

我们可以看到UMA下底层的分配函数\_\_\_alloc\_bootmem\_nopanic与NUMA下的函数\_\_\_alloc\_bootmem\_node\_nopanic实现方式基本类似. 参数也基本相同

| 参数 | 描述 |
|:-----:|:-----|
| pgdat | 要分配的结点， 在UMA结构中, 它被缺省掉了, 因此其默认值是contig\_page\_data |
| size | 要分配的内存区域大小
| align | 要求对齐的字节数. 如果分配的空间比较小, 就用SMP\_CACHE\_BYTES， 它一般是硬件一级高速缓存的对齐方式, 而PAGE_SIZE则表示要在页边界对齐 |
| goal | 最佳分配的起始地址, 一般设置(normal)BOOTMEM\_LOW\_LIMIT / (low)ARCH\_LOW\_ADDRESS\_LIMIT 

## 5.3 \_\_alloc\_memory\_core进行内存分配

| 函数 | 描述 | 定义 |
|:-----:|:-----:|:-----:|
| alloc\_bootmem\_bdata |  | [mm/bootmem.c?v=4.7, line 500](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L500) |
| alloc\_bootmem\_core |    | [mm/bootmem.c, line 607](http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L607) |

\_\_alloc\_memory\_core函数的功能相对而言很广泛(在启动期间不需要太高的效率), 该函数基于最先适配算法, 但是该分配器不仅可以分配整个内存页, 还能分配页的一部分. 它遍历所有的bootmem list然后找到一个合适的内存区域, 然后通过 alloc\_bootmem\_bdata来完成分配

该函数主要执行如下操作

- list\_for\_each\_entry从goal开始扫描为图, 查找满足分配请求的空闲内存区

- 然后通过alloc\_bootmem\_bdata完成内存的分配
	
1. 如果目标页紧接着上一次分配的页即last\_end\_off, 则内核会判断所需的内存(包括对齐数据所需的内存)是否能够在上一页分配或者从上一页开始分配

2. 新分配的页在位图中对应位置设置为1,, 如果该页未完全分配, 则相应的偏移量保存在bootmem\_data->last\_end\_off中; 否则, 该值设为0

# 6 bootmem释放内存

内核提供了free\_bootmem函数来释放内存

它需要两个参数：需要释放的内存区的起始地址和长度。不出意外，NUMA系统上等价函数的名称为free\_bootmem\_node，它需要一个额外的参数来指定结点

```cpp
//  http://lxr.free-electrons.com/source/mm/bootmem.c?v=4.7#L422
void free_bootmem(unsigned long addr, unsigned long size);
void free_bootmem_node(pg_data_t *pgdat, unsigned long addr, unsigned long size);
```

# 7 停用bootmem

在系统初始化进行到伙伴系统分配器能够承担内存管理的责任后，必须停用bootmem分配器，毕竟不能同时用两个分配器管理内存。在UMA和NUMA系统上，停用是由[**free\_all\_bootmem**](http://lxr.free-electrons.com/source/mm/bootmem.c#L247)完成。在伙伴系统建立之后，**特定于体系结构的初始化代码**需要调用这个函数

首先扫描bootmem分配器的**页位图**，释放**每个未用的页**。到**伙伴系统的接口**是\_\_free\_pages\_bootmem函数，该函数对每个空闲页调用。该函数内部依赖于标准函数\_\_free\_page。它使得这些页并入伙伴系统的数据结构，在其中作为空闲页管理，可用于分配。

在页位图已经完全扫描之后，它占据的内存空间也必须释放。此后，只有伙伴系统可用于内存分配。

# 8 链接

[Linux节点和内存管理区的初始化](http://blog.csdn.net/vanbreaker/article/details/7554977)

[linux内存管理之初始化zonelists](http://blog.csdn.net/yuzhihui_no1/article/details/50759567)

[Linux-2.6.32 NUMA架构之内存和调度](http://www.cnblogs.com/zhenjing/archive/2012/03/21/linux_numa.html)

[Linux节点和内存管理区的初始化](http://www.linuxidc.com/Linux/2012-05/60230.htm)

[(内存管理)bootmem没了?](http://bbs.chinaunix.net/thread-4141073-1-1.html)

[linux_3.0.1分析3--arm 内存初始化2-qemu ](http://blog.chinaunix.net/uid-26009923-id-3860465.html)

[Linux内存模型之bootmem分配器](http://www.linuxidc.com/Linux/2012-02/53139.htm)

[Vi Linux内存 之 bootmem分配器（一） ](http://blog.chinaunix.net/uid-7588746-id-1629805.html)

[linux-3.2.36内核启动1-启动参数（arm平台 启动参数的获取和处理，分析setup_arch）](http://blog.csdn.net/xxxxxlllllxl/article/details/12091667)