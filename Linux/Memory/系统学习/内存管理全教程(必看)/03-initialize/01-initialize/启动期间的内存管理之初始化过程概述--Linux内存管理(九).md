[TOC]

- 1 前景回顾
    - 1.1 Linux内存管理的层次结构
    - 1.2 内存结点pg\_data\_t
    - 1.3 物理内存区域
    - 1.4 物理页帧
    - 1.5 今日内容(启动过程中的内存初始化)
- 2 第一阶段(启动过程中的内存管理)
    - 2.1 引导内存分配器bootmem
    - 2.2 memblock内存分配器
    - 2.3 两者的区别与联系
    - 2.4 memblock的初始化(arm64架构)
- 3 第二阶段(初始化buddy内存管理)
    - 3.1 初始化流程
    - 3.2 paging\_init初始化分页机制
    - 3.3 虚拟地址空间(以x86\_32位系统为例)
    - 3.4 bootmem\_init初始化内存的基础数据结构(结点pg_data, 内存域zone, 页面page)
    - 3.5 build\_all\_zonelists初始化每个内存节点的zonelists
- 4 总结
    - 4.1 start\_kernel启动流程
    - 4.2 体系结构相关的初始化工作setup\_arch
    - 4.3 bootmem\_init初始化内存的基础数据结构(结点pg\_data, 内存域zone, 页面page)
    - 4.4 build\_all\_zonelists初始化每个内存节点的zonelists

在内存管理的上下文中, 初始化(initialization)可以有多种含义.在许多CPU上,必须**显式设置**适用于Linux内核的**内存模型**.在x86\_32上需要切换到保护模式, 然后内核才能检测到可用内存和寄存器.

而我们今天要讲的boot阶段就是系统初始化阶段使用的内存分配器. 

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

- 最后**页帧(page frame**)代表了系统内存的最小单位, 堆内存中的每个页都会创建一个struct page的一个实例. 传统上，把内存视为连续的字节，即内存为字节数组，内存单元的编号(地址)可作为字节数组的索引. 分页管理时，将若干字节视为一页，比如4K byte. 此时，内存变成了连续的页，即内存为页数组，每一页物理内存叫页帧，以页为单位对内存进行编号，该编号可作为页数组的索引，又称为页帧号.

## 1.2 内存结点pg\_data\_t

在Linux中引入一个数据结构`struct pglist_data` ，来描述一个node，定义在[`include/linux/mmzone.h`](http://lxr.free-electrons.com/source/include/linux/mmzone.h#L630) 文件中。（这个结构被typedef pg\_data\_t）。

- 对于NUMA系统来讲，整个系统的内存由一个[**node\_data**](http://lxr.free-electrons.com/source/arch/s390/numa/numa.c?v=4.7#L23)的pg\_data\_t**指针数组**来管理

- 对于UMA系统，使用struct pglist\_data contig\_page\_data ，作为系统唯一的node管理所有的内存区域。（UMA系统中中只有一个node）

可以使用NODE\_DATA(node\_id)来查找系统中编号为node\_id的结点, 而UMA结构下由于只有一个结点, 因此该宏总是返回全局的contig\_page\_data, 而与参数node\_id无关.

**NODE\_DATA(node\_id)查找编号node\_id的结点pg\_data\_t信息** 参见[NODE\_DATA的定义](http://lxr.free-electrons.com/ident?v=4.7;i=NODE_DATA)

```cpp
extern struct pglist_data *node_data[];
#define NODE_DATA(nid)          (node_data[(nid)])
```

在UMA结构的机器中, 只有一个node结点即contig\_page\_data,此时NODE\_DATA直接指向了全局的contig\_page\_data,而与node的编号nid无关,参照[include/linux/mmzone.h?v=4.7, line 858](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L858)

```cpp
extern struct pglist_data contig_page_data;
#define NODE_DATA(nid)          (&contig_page_data)
```

## 1.3 物理内存区域

因为实际的计算机体系结构有硬件的诸多限制, 这限制了页框可以使用的方式. 尤其是, Linux内核必须处理80x86体系结构的两种硬件约束.

- ISA总线的直接内存存储DMA处理器有一个严格的限制 : 他们只能对RAM的前16MB进行寻址

- 在具有大容量RAM的现代32位计算机中, CPU不能直接访问所有的物理地址, 因为线性地址空间太小, 内核不可能直接映射所有物理内存到线性地址空间, 我们会在后面典型架构(x86)上内存区域划分详细讲解x86\_32上的内存区域划分

因此Linux内核对不同区域的内存需要采用不同的管理方式和映射方式, 因此内核将物理地址或者成用zone\_t表示的不同地址区域

对于x86\_32的机器，管理区(内存区域)类型如下分布

| 类型 | 区域 |
| :------- | :---- |
| ZONE_DMA | 0~15MB |
| ZONE_NORMAL | 16MB~895MB |
| ZONE_HIGHMEM | 896MB~物理内存结束 |

## 1.4 物理页帧

内核把物理页作为内存管理的基本单位.尽管处理器的最小可寻址单位通常是字,但是,内存管理单元MMU通常以页为单位进行处理.因此，从虚拟内存的上来看，页就是最小单位.

页帧代表了系统内存的最小单位, 对内存中的每个页都会创建struct page的一个实例. 内核必须要保证page结构体足够的小，否则仅struct page就要占用大量的内存.

内核用[struct  page(include/linux/mm_types.h?v=4.7, line 45)](http://lxr.free-electrons.com/source/include/linux/mm_types.h?v4.7#L45)结构表示系统中的每个物理页.

出于节省内存的考虑，struct page中使用了大量的联合体union.

`mem_map`是一个struct page的数组，管理着系统中所有的物理内存页面。在系统启动的过程中，创建和分配mem\_map的内存区域, mem\_map定义在[mm/page\_alloc.c?v=4.7, line 6691](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L6691)

UMA体系结构中，free\_area\_init函数在系统唯一的struct node对象contig\_page\_data中node\_mem\_map成员赋值给全局的mem\_map变量

## 1.5 今日内容(启动过程中的内存初始化)

在初始化过程中, 还必须建立内存管理的数据结构,以及很多事务.因为内核在内存管理完全初始化之前就需要使用内存.在系统启动过程期间,使用了额外的简化的**内存管理模块**,然后在初始化完成后,将**旧的模块丢弃**掉.

因此我们可以把linux内核的**内存管理分三个阶段**。

| 阶段 | 起点 | 终点 | 描述 |
|:-----|:-----|:-----|:-----|
| 第一阶段 | 系统启动 | bootmem或者memblock初始化完成 | 此阶段只能使用**memblock\_reserve函数**分配内存，**早期内核**中使用init\_bootmem\_done = 1标识此阶段结束 |
| 第二阶段 | bootmem或者memblock初始化完 | buddy完成前 | **引导内存分配器bootmem**或者**memblock**接受内存的管理工作, **早期内核**中使用mem\_init\_done = 1标记此阶段的结束 |
| 第三阶段 | buddy初始化完成 | 系统停止运行 | 可以用**cache和buddy分配**内存 |

**系统启动过程中的内存管理**

首先我们来看看start\_kernel是如何初始化系统的,start\_kerne定义在[init/main.c?v=4.7, line 479](http://lxr.free-electrons.com/source/init/main.c?v=4.7#L479)

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

# 2 第一阶段(启动过程中的内存管理)

内存管理是操作系统资源管理的重点,但是在**操作系统初始化的初期**,操作系统只是获取到了**内存的基本信息**,但是内存管理的**数据结构**都**没有建立**,而我们这些**数据结构创建的过程**本身就是一个**内存分配的过程**,那么就出现一个问题

我们还**没有**一个**内存管理器**去负责**分配和回收**内存,而我们又**不可能**将**所有的内存信息都静态创建并初始化**,那么我们怎么分配内存管理器所需要的内存呢? 现在我们进入了一个先有鸡还是先有蛋的怪圈,这种问题的**一般解决方法**是,我们先实现一个**满足要求**的但是**可能效率不高**的笨家伙(内存管理器),用它来负责系统**初始化初期**的内存管理,**最重要的**,用它来**初始化我们内存的数据结构**,直到我们真正的内存管理器被初始化完成并能投入使用,我们将旧的内存管理器丢掉

即因此在系统启动过程期间,内核使用了一个额外的简化形式的内存管理模块早期的**引导内存分配器(boot memory allocator--bootmem分配器)**或者**memblock**, 用于在启动阶段早期分配内存, 而在**系统初始化完成**后, 该**分配器被内核抛弃**, 然后初始化了一套新的更加完善的内存分配器.

## 2.1 引导内存分配器bootmem

在启动过程期间, 尽管内存管理尚未初始化,但是内核仍然需要分配内存以创建各种数据结构,早期的内核中负责初始化阶段的内存分配器称为**引导内存分配器(boot memory allocator--bootmem分配器)**,在耳熟能详的伙伴系统建立前内存都是利用分配器来分配的，伙伴系统框架建立起来后，bootmem会过度到伙伴系统.显然,对该内存分配器的需求集中于**简单**性方面,而不是性能和通用性,它仅用于初始化阶段.因此内核开发者决定实现一个**最先适配(first-first)分配器**用于在启动阶段管理内存. 这是可能想到的最简单的方式.

**引导内存分配器(boot memory allocator--bootmem分配器)**基于**最先适配(first-first)分配器**的原理(这儿是很多系统的内存分配所使用的原理),使用**一个位图来管理页**,以**位图**代替原来的**空闲链表结构**来表示**存储空间**,位图的比特位的数目与系统中物理内存页面数目相同. 若**位图中某一位是1**,则标识**该页面**已经**被分配**(已用页),否则表示未被占有(未用页).

在需要分配内存时, 分配器逐位的**扫描位图**,直至找到一个能提供**足够连续页(空间连续！！！**）的位置,即所谓的**最先最佳(first-best)或最先适配**位置.该分配机制通过记录上一次分配的**页面帧号(PFN**)结束时的**偏移量**来实现分配**大小小于一页的空间**, 连续的小的**空闲空间**将被合并存储在**一页**上.

即使是初始化用的最先适配分配器也必须使用一些数据结构存,内核为系统中**每一个结点（每个node！！！**）都提供了一个**struct bootmem\_data结构的实例**,用于bootmem的内存管理. 它含有引导内存分配器给结点分配内存时所需的信息.当然,这时候内存管理还没有初始化, 因而该**结构所需的内存**是**无法动态分配**的,必须在**编译时分配给内核(！！！**）.

在**UMA系统**上该分配的实现**与CPU无关**,而NUMA系统内存结点与CPU相关联,因此采用了特定体系结构的解决方法.

bootmem\_data的结构定义在[include/linux/bootmem.h?v=4.7, line 28](http://lxr.free-electrons.com/source/include/linux/bootmem.h?v=4.7#L28)

关于引导内存分配器的具体内容, 请参见另外一篇文章

## 2.2 memblock内存分配器

但是bootmem也有很多问题.最明显的就是**外碎片**的问题,因此内核维护了**memblock内存分配器**,同时用memblock实现了一份**bootmem相同的兼容API**,即nobootmem,Memblock以前被定义为Logical Memory Block(逻辑内存块), 但根据[Yinghai Lu的补丁](https://lkml.org/lkml/2010/7/13/68), 它被重命名为memblock. 并最终替代bootmem成为初始化阶段的内存管理器

关于引导内存分配器的具体内容, 请参见另外一篇文章

## 2.3 两者的区别与联系

bootmem是通过**位图来管理**，位图存在**低地址段**,而memblock是在**高地址管理内存**, 维护**两个链表**, 即**memory和reserved**

**memory链表**维护系统的**内存信息**(在初始化阶段**通过bios获取**的),对于任何**内存分配**,先去**查找memory链表**,然后**在reserve链表上记录**(新增一个节点，或者合并)

1. 两者都可以分配**小于一页的内存**；

2. 两者都是**就近查找可用的内存**， bootmem是从低到高找，memblock是从高往低找；

在boot传递给kernel memory bank相关信息后，kernel这边会以**memblcok的方式保存**这些信息，当buddy system 没有起来之前，在kernel中也是要有一套机制来管理memory的申请和释放.

Kernel可以选择nobootmem或者bootmem来在buddy system起来之前管理memory.这两种机制对**提供的API是一致的**，因此对用户是透明的

参见[mm/**Makefile**](http://lxr.free-electrons.com/source/mm/Makefile#L44)

```cpp
ifdef CONFIG_NO_BOOTMEM
	obj-y           += nobootmem.o
else
	obj-y           += bootmem.o
endif
```

由于接口是一致的, 那么他们共同使用一份头文件

| 头文件 | bootmem接口 | nobootmem接口 |
|:-----:|:-----------:|:------------:|
| [include/linux/bootmem.h](http://lxr.free-electrons.com/source/include/linux/bootmem.h) | [mm/bootmem.c](http://lxr.free-electrons.com/source/mm/bootmem.c) | [mm/nobootmem.c](http://lxr.free-electrons.com/source/mm/nobootmem.c) |

## 2.4 memblock的初始化(arm64架构)

前面我们的内核从**start\_kernel**开始,进入**setup\_arch**(),并完成了**早期内存分配器的初始化和设置**工作.

```cpp
void __init setup_arch(char **cmdline_p)
{
	/*  初始化memblock  */
	arm64_memblock_init( );

	/*  分页机制初始化  */
	paging_init();

	bootmem_init();
}
```

| 流程 | 描述 |
|:---:|:----:|
| [arm64_memblock_init](http://lxr.free-electrons.com/source/arch/arm64/kernel/setup.c?v=4.7#L229) | 初始化memblock内存分配器 |
| [paging_init](http://lxr.free-electrons.com/source/arch/arm64/mm/mmu.c?v=4.7#L538) | 初始化分页机制 |
| [bootmem_init](http://lxr.free-electrons.com/source/arch/arm64/mm/init.c?v=4.7#L306) | 初始化内存管理 |

其中arm64\_memblock\_init就完成了**arm64架构**下的memblock的初始化

# 3	第二阶段(初始化buddy内存管理)

## 3.1 初始化流程

下面我们就以**arm64架构**来分析bootmem初始化内存**结点**和**内存域**的过程, 在讲解的过程中我们会兼顾的考虑arm64架构下的异同

- 首先内核从[**start\_kernel**](http://lxr.free-electrons.com/source/init/main.c?v=4.7#L505)开始启动

- 然后进入**体系结构相关**的设置部分[**setup\_arch**](http://lxr.free-electrons.com/source/arch/arm/kernel/setup.c?v=4.7#L1073),开始**获取并设置**指定**体系结构**的一些**物理信息**,而arm64架构下则对应着[arch/arm64/kernel/setup.c](http://lxr.free-electrons.com/source/arch/arm64/kernel/setup.c?v=4.7#L229)

- 在**setup\_arch**函数内, 通过**paging\_init**函数初始化了**分页机制**和**页表**的信息

- 接着**paging\_init**函数通过**调用**[**bootmem\_init**](http://lxr.free-electrons.com/source/arch/arm/mm/mmu.c#L1642)开始进行初始化工作

**arm64**在整个初始化的流程上并没有什么不同, 但是有细微的差别

- 由于**arm**是在**后期才开始加入了MMU内存管理单元**的,因此内核必须实现**mmu**和**nonmmu**两套**不同的代码**,这主要是体现在分页机制的不同上,因而paging\_init分别定义了[arch/arm/mm/nommu.c](http://lxr.free-electrons.com/source/arch/arm/mm/nommu.c?v=4.7#L311)和[arch/arm/mm/mmu.c](http://lxr.free-electrons.com/source/arch/arm/mm/mmu.c?v=4.7#L1623)两个版本,但是它们**均调用**了**bootmem_init**来完成初始化

- 也是因为上面的原因, arm上**paging\_init**有**两份代码**([mmu](http://lxr.free-electrons.com/source/arch/arm/mm/mmu.c?v=4.7#L162)和[nonmmu](http://lxr.free-electrons.com/source/arch/arm/mm/nommu.c?v=4.7#L311)),为了降低**代码的耦合性**,arm通过setup\_arch调用**paging\_init**函数,**paging\_init**进一步**调用了bootmem\_init**来完成,而arm64上不存在这样的问题,则在[setup\_arch中顺序的先用paging\_init](http://lxr.free-electrons.com/source/arch/arm64/kernel/setup.c?v=4.7#L266)初始化了页表, 然后[**setup\_arch又调用bootmem\_init**](http://lxr.free-electrons.com/source/arch/arm64/kernel/setup.c?v4.7#L271)来完成了bootmem的初始化

## 3.2 paging\_init初始化分页机制

paging\_init负责建立**只能用于内核的页表**,**用户空间**是**无法访问**的.这对管理普通应用程序和内核访问内存的方式，有深远的影响

因此在仔细考察其实现之前，很重要的一点是解释该函数的目的。

在x86\_32系统上内核通常将总的4GB可用虚拟地址空间按3:1的比例划分给用户空间和内核空间, 虚拟地址空间的低端3GB用于用户状态应用程序,而高端的1GB则专用于内核.尽管在分配内核的虚拟地址空间时,当前系统上下文是不相干的,但每个进程都有自身特定的地址空间.

这些划分主要的动机如下所示

- 在用户**应用程序**的执行**切换到核心态**时（这总是会发生，例如在使用系统调用或发生周期性的时钟中断时），**内核**必须装载在一个**可靠的环境**中。因此有必要将地址空间的一部分分配给内核专用.

- **物理内存页**则映射到**内核地址空间的起始处**，以便内核直接访问，而无需复杂的页表操作.

如果所有物理内存页都映射到用户空间进程能访问的地址空间中,如果在系统上有**几个应用程序在运行**,将导致严重的**安全问题**.每个应用程序都能够读取和修改其他进程在物理内存中的内存区. 显然必须不惜任何代价防止这种情况出现.

虽然用于**用户层进程**的**虚拟地址部分**随进程切换而**改变**，但是**内核部分总是相同的**

## 3.3 虚拟地址空间(以x86\_32位系统为例)

出于内存保护等一系列的考虑,内核将整个进程的虚拟运行空间划分为内核虚拟运行空间和内核虚拟运行空间

![虚拟地址空间](./images/vmarea_space.jpg)

按3:1的比例划分地址空间,只是约略反映了内核中的情况，**内核地址空间**作为内核的**常驻虚拟地址空间**, 自身又分为各个段

![内核空间](./images/kernel_space.jpg)

地址空间的**第一段**用于将系统的**所有物理内存页**映射到**内核的虚拟地址空间**中。由于内核地址空间从偏移量0xC0000000开始，即经常提到的3 GB，每个虚拟地址x都对应于物理地址(x \- 0xC0000000)，因此这是一个简单的线性平移。

**直接映射区域**从**0xC0000000到high\_memory**地址，high\_memory准确的数值稍后讨论。第1章提到过，这种方案有一问题。由于内核的虚拟地址空间只有1GiB，最多只能映射1 GiB物理内存。IA-32系统（没有PAE）最大的内存配置可以达到4GiB，引出的一个问题是，如何处理剩下的内存？

这里有个坏消息。如果**物理内存超过896MiB**，则**内核无法直接映射全部物理内存**。该值甚至比此前提到的最大限制1GiB还小，因为内核必须保留地址空间**最后的128MiB用于其他目的**，我会稍后解释。将这128MiB加上直接映射的896MiB内存，则得到内核虚拟地址空间的总数为1024MiB=1GiB。内核使用两个经常使用的缩写normal和highmem，来区分是否可以直接映射的页帧。

内核地址空间的最后128 MiB用于何种用途呢？该部分有3个用途。

- **虚拟内存中连续**、但**物理内存中不连续**的内存区，可以在**vmalloc区域**分配。该机制**通常用于用户过程**，内核自身会试图尽力**避免非连续的物理地址**。内核通常会成功，因为大部分大的内存块都在启动时分配给内核，那时内存的碎片尚不严重。但在已经运行了很长时间的系统上，在内核需要物理内存时，就可能出现可用空间不连续的情况。此类情况，主要出现在**动态加载模块**时

- **持久映射**用于将高端内存域中的**非持久页（物理页！！！）映射到内核**中

- **固定映射**是与**物理地址**空间中的**固定页**关联的**虚拟地址空间项**，但具体关联的**页帧（物理页**）可以自由选择。它与通过固定公式与物理内存关联的**直接映射页**相反，**虚拟固定映射地址**与**物理内存**位置之间的**关联可以自行定义**，关联建立后内核总是会注意到的

同样我们的[**用户空间**](http://www.360doc.com/content/14/1020/21/19947352_418512226.shtml), 也被划分为几个段, 包括从高地址到低地址分别为 :

![进程的虚拟地址空间](./images/user_space.jpg)

| 区域 | 存储内容 |
|:---:|:------:|
|  栈   | 局部变量, 函数参数, 返回地址等 |
|  堆   | 动态分配的内存 |
| BSS段 | 未初始化或初值为0的全局变量和静态局部变量|
| 数据段 | 一初始化且初值非0的全局变量和静态局部变量|
| 代码段 | 可执行代码, 字符串面值, 只读变量 |

## 3.4 bootmem\_init初始化内存的基础数据结构(结点pg\_data, 内存域zone, 页面page)

在paging\_init之后,系统的页帧已经建立起来,然后通过bootmem\_init中,系统开始完成bootmem的初始化工作.

**不同的体系**结构bootmem\_init的实现, 没有很大的区别, 但是在**初始化**的过程中, 其中的很多函数, 依据系统是NUMA还是UMA结构则有不同的定义

bootmem\_init函数的实现如下

| 函数实现 | arm | arm64 |
|:---:|:---:|:-----:|
| bootmem_init | [arch/arm/mm/init.c, line 282](http://lxr.free-electrons.com/source/arch/arm/mm/init.c?v=4.7#L282) | [arch/arm64/mm/init.c, line 306](http://lxr.free-electrons.com/source/arch/arm64/mm/init.c?v=4.7#L306) |

## 3.5 build\_all\_zonelists初始化每个内存节点的zonelists

内核setup\_arch的最后通过bootmem\_init中完成了**内存数据结构的初始化**(包括**内存结点pg\_data\_t**,内存**管理域zone**和**页面信息page**),数据结构已经基本准备好了,在后面为内存管理做得一个准备工作就是将**所有节点的管理区（所有的节点pg\_data\_t的zone！！！**）都链入到**zonelist**中，便于后面内存分配工作的进行.

内存节点pg\_data\_t中将内存节点中的内存区域zone按照**某种组织层次**（可配置！！！）存储在一个zonelist中, 即pglist\_data->node\_zonelists成员信息

```cpp
//  http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L626
typedef struct pglist_data
{
	struct zone node_zones[MAX_NR_ZONES];
	struct zonelist node_zonelists[MAX_ZONELISTS];
}
```

内核定义了内存的一个层次结构关系, 首先试图分配廉价的内存，如果失败，则根据访问速度和容量，逐渐尝试分配更昂贵的内存.

**高端内存最廉价**, 因为**内核**没有任何部分**依赖于从该内存域分配的内存**,如果高端内存用尽,对内核没有副作用, 所以优先分配高端内存

普通内存域的情况有所不同, 许多内核数据结构必须保存在该内存域, 而不能放置到高端内存域, 因此如果普通内存域用尽, 那么内核会面临内存紧张的情况

DMA内存域最昂贵，因为它用于外设和系统之间的数据传输。

举例来讲，如果内核指定想要分配高端内存域。它首先在当前结点的高端内存域寻找适当的空闲内存段，如果失败，则查看该结点的普通内存域，如果还失败，则试图在该结点的DMA内存域分配。如果在3个本地内存域都无法找到空闲内存，则查看其他结点。这种情况下，备选结点应该尽可能靠近主结点，以最小化访问非本地内存引起的性能损失。

# 4 总结

## 4.1 start\_kernel启动流程

```cpp
start_kernel()
    |---->page_address_init()
    |     考虑支持高端内存
    |     业务：初始化page_address_pool链表；
    |          将page_address_maps数组元素按索引降序插入
    |          page_address_pool链表; 
    |          初始化page_address_htable数组.
    | 
    |---->setup_arch(&command_line);   
    |     初始化特定体系结构的内容
    	|---->arm64_memblock_init( );  [参见memblock和bootmem]
        |     初始化引导阶段的内存分配器memblock
        |
        |---->paging_init();  [参见分页机制初始化paging_init]
        |     分页机制初始化
        |
        |---->bootmem_init();  [与build_all_zonelist共同完成内存数据结构的初始化]
        |       初始化内存数据结构包括内存节点和内存域
        |
    |---->setup_per_cpu_areas();
    |     为per-CPU变量分配空间
    |
    |---->build_all_zonelist();  [bootmem_init初始化数据结构, 该函数初始化zonelists]
    |     为系统中的zone建立后备zone的列表.
    |     所有zone的后备列表都在
    |     pglist_data->node_zonelists[0]中;
    |
    |     期间也对per-CPU变量boot_pageset做了初始化. 
    |
    |---->page_alloc_init()
         |---->hotcpu_notifier(page_alloc_cpu_notifier, 0);
         |     不考虑热插拔CPU 
         |
    |---->pidhash_init()
    |     详见下文.
    |     根据低端内存页数和散列度，分配hash空间，并赋予pid_hash
    |
    |---->vfs_caches_init_early()
          |---->dcache_init_early()
          |     dentry_hashtable空间，d_hash_shift, h_hash_mask赋值；
          |     同pidhash_init();
          |     区别:
          |         散列度变化了（13 - PAGE_SHIFT）;
          |         传入alloc_large_system_hash的最后参数值为0;
          |
          |---->inode_init_early()
          |     inode_hashtable空间，i_hash_shift, i_hash_mask赋值；
          |     同pidhash_init();
          |     区别:
          |         散列度变化了（14 - PAGE_SHIFT）;
          |         传入alloc_large_system_hash的最后参数值为0;
          |
```

## 4.2 体系结构相关的初始化工作setup_arch

Linux内核启动函数start\_kernel()，这里会调用setup\_arch()完成与体系结构相关的一系列初始化工作，其中就包括各种内存的初始化工作，如内存图的建立、管理区的初始化等等。

ARM64的setup\_arch

```cpp
//arm64
setup_arch(char **cmdline_p)
    |---->arm64_memblock_init( );
    |     初始化引导阶段的内存分配器memblock
	|
    |
    |---->paging_init();
    |     分页机制初始化
	|
    |
    |---->bootmem_init();
    |		初始化内存数据结构包括内存节点和内存域
}
```

x86的setup\_arch

```cpp
// arch/x86/kernel/setup.c

void __init setup_arch(char **cmdline_p)
{
    /* ...... */
    x86_init.oem.arch_setup();
    e820__memory_setup();
    parse_setup_data();
    
    /* ...... */
    
    /* update the e820_saved too */
    e820__reserve_setup_data();
    
    max_pfn = e820__end_of_ram_pfn();
    
#ifdef CONFIG_X86_32
    /* max_low_pfn在这里更新 */
    find_low_pfn_range(); /* 找出低端内存的最大页帧号 */
#else
    check_x2apic();
    
    /* How many end-of-memory variables you have, grandma! */
    /* need this before calling reserve_initrd */
    if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
        max_low_pfn = e820__end_of_low_ram_pfn();
    else
        max_low_pfn = max_pfn;
 
    high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
#endif

    /*页表缓冲区申请*/
    early_alloc_pgt_buf();
    
    /* e820__memblock_setup()之前需要 */
    reserve_brk();
    
    memblock_set_current_limit(ISA_END_ADDRESS);
    
    /* memblock建立 */
    e820__memblock_setup();
    
    /* 初始化低端内存和高端内存固定映射区的页表 */
    init_mem_mapping();
    
    /* 初始化内存分配器 */
    initmem_init();
    
    /* 建立高端内存永久映射区的页表并获取固定映射区的临时映射区页表 */
    x86_init.paging.pagetable_init();
}
```

几乎所有的内存初始化工作都是在setup_arch()中完成的，主要的工作包括：

（1）建立内存：e820\_\_memory\_setup()，排序整理e820提供的内存，对应原来的**setup\_memory\_map**，可以参见教程Linux-3.14.12的《8. 系统启动阶段的memblock算法（一）》;

（2）调用e820\_end\_of\_ram\_pfn()找出最大可用页帧号max\_pfn，调用find\_low\_pfn\_range()找出低端内存区的最大可用页帧号max\_low\_pfn。

（2）初始化低端内存和高端内存固定映射区的页表：init\_memory\_mapping()；

（3）初始化内存分配器：initmem\_init()；

（4）建立高端内存永久映射区的页表并获取固定映射区的临时映射区页表：x86\_init.paging.pagetable\_init()。

**x86的setup\_arch可以参见教程Linux-3.14.12**

## 4.3 bootmem\_init初始化内存的基础数据结构(结点pg\_data, 内存域zone, 页面page)

```cpp
bootmem_init(void)
    |---->min = PFN_UP(memblock_start_of_DRAM());
    |---->max = PFN_DOWN(memblock_end_of_DRAM());
    |
    |
    |---->arm64_numa_init();
    |     支持numa架构
    |---->arm64_numa_init();
    |     支持numa架构
    |
    |
    |---->zone_sizes_init(min, max);
        来初始化节点和管理区的一些数据项
        |
        |---->free_area_init_node
        |   初始化内存节点
        |
        |
            |---->free_area_init_core初始化zone
                |
                |
                |---->memmap_init初始化page页面
                |
                |
    |
    |---->memblock_dump_all();
    |   初始化完成, 显示memblock的保留的所有内存信息
```

## 4.4 build\_all\_zonelists初始化每个内存节点的zonelists

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