
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [0 内存虚拟化](#0-内存虚拟化)
* [1 概述](#1-概述)
* [2 相关数据结构](#2-相关数据结构)
	* [2.1 AddressSpace](#21-addressspace)
	* [2.2 MemoryRegion](#22-memoryregion)
	* [2.3 RAMBlock](#23-ramblock)
	* [2.4 FlatView](#24-flatview)
	* [2.5 MemoryRegionSection](#25-memoryregionsection)
	* [2.6 MemoryListener](#26-memorylistener)
	* [2.7 AddressSpaceDispatch](#27-addressspacedispatch)
* [3 初始化流程](#3-初始化流程)
	* [3.1 memory\_map\_init: 全局memory space和io space初始化](#31-memory_map_init-全局memory-space和io-space初始化)
	* [3.2 pc\_memory\_init](#32-pc_memory_init)
		* [3.2.1 pc.ram分配](#321-pcram分配)
		* [3.2.2 两个mr alias: ram\_below\_4g和ram\_above\_4g](#322-两个mr-alias-ram_below_4g和ram_above_4g)
* [4 内存的提交](#4-内存的提交)

<!-- /code_chunk_output -->

# 0 内存虚拟化

内存虚拟化就是为虚拟机提供内存，使得虚拟机能够像在物理机上正常工作，这需要虚拟化软件为虚拟机展示一种物理内存的假象，内存虚拟化是虚拟化技术中关键技术之一。qemu+kvm的虚拟化方案中，内存虚拟化是由qemu和kvm共同完成的。qemu的虚拟地址作为guest的物理地址，一句看似轻描淡写的话幕后的工作确实非常多，加上qemu本身可以独立于kvm，成为一个完整的虚拟化方案，所以其内存虚拟化更加复杂。

本文主要介绍qemu在内存虚拟化方面的工作,之后的文章会介绍内存kvm方面的内存虚拟化。

# 1 概述

内存虚拟化就是要让**虚拟机**能够**无缝的访问内存**，这个内存哪里来的，**qemu**的**进程地址空间**分出来的。

有了ept之后，CPU在vmx non\-root状态的时候进行内存访问会再做一个ept转换。在这个过程中，**qemu**扮演的角色。

1. 首先需要去**申请内存用于虚拟机**； 

2. 需要将虚拟1中**申请的地址**的**虚拟地址**与**虚拟机**的对应的**物理地址**告诉给kvm，就是指定**GPA\->HVA**的映射关系；

3. 需要组织一系列的数据结构去管理控制内存虚拟化，比如，设备注册需要分配物理地址，虚拟机退出之后需要根据地址做模拟等等非常多的工作，由于qemu本身能够支持tcg模式的虚拟化，会显得更加复杂。

首先明确**内存虚拟化**中**QEMU**和**KVM**工作的分界。

KVM的**ioctl**中，**设置虚拟机内存**的为**KVM\_SET\_USER\_MEMORY\_REGION**，我们看到这个ioctl需要传递的参数是:

```c
/* for KVM_SET_USER_MEMORY_REGION */
struct kvm_userspace_memory_region {
	__u32 slot;
	__u32 flags;
	__u64 guest_phys_addr;
	__u64 memory_size; /* bytes */
	__u64 userspace_addr; /* start of the userspace allocated memory */
};
```

这个**ioctl**主要就是**设置GPA到HVA的映射**。看似简单的工作在qemu里面却很复杂，下面逐一剖析之。

# 2 相关数据结构

## 2.1 AddressSpace

首先，

**所有的CPU架构**都有**内存地址空间**, 有些CPU架构又有一个**IO地址空间**。它们在QEMU中被表示为**AddressSpace数据结构**.

qemu中用**AddressSpace**用来表示**CPU/设备**看到的**内存**.

```c
// include/exec/memory.h
struct AddressSpace {
    /* All fields are private. */
    struct rcu_head rcu;
    char *name;
    // MR树(多个MR)
    MemoryRegion *root;

    /* Accessed via RCU.  */
    //AddressSpace的一张平面视图，它是AddressSpace所有正在使用的MemoryRegion的集合，这是从CPU的视角来看到的。
    struct FlatView *current_map;

    int ioeventfd_nb;
    struct MemoryRegionIoeventfd *ioeventfds;
    QTAILQ_HEAD(, MemoryListener) listeners;
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};
```

AddressSpace下面**root**及其子树形成了**一个虚拟机**的**物理地址**

## 2.2 MemoryRegion

一个AddressSpace下面包含**多个MemoryRegion**，这些MemoryRegion结构通过**树**连接起来，树的根是AddressSpace的root域。

**结构体MemoryRegion**是联系**guest物理地址空间**和描述**真实内存的RAMBlocks(宿主机虚拟地址**)之间的桥梁。

```c
// include/exec/memory.h
struct MemoryRegion {
    Object parent_obj;

    /* All fields are private - violators will be prosecuted */

    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool nonvolatile;
    bool rom_device;
    bool flush_coalesced_mmio;
    bool global_locking;
    //表示哪种dirty map被使用，共有三种
    uint8_t dirty_log_mask;
    bool is_iommu;
    // 分配的实际内存HVA
    RAMBlock *ram_block;
    Object *owner;
    // 与MemoryRegion相关的操作
    const MemoryRegionOps *ops;
    void *opaque;
    MemoryRegion *container;
    Int128 size;
    //在AddressSpace中的地址
    hwaddr addr;
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    //是否是ram内存, 区别于rom只读
    bool ram_device;
    //如果为true，表示已经通知kvm使用这段内存
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;
    hwaddr alias_offset;
    int32_t priority;
    QTAILQ_HEAD(, MemoryRegion) subregions;
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(, CoalescedMemoryRange) coalesced;
    //MemoryRegion的名字,调试时使用
    const char *name;
    //IOevent文件描述符的管理
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
};
```

MemoryRegion有**多种类型**，可以表示一段**ram**、**rom**、**MMIO**、**alias**.

alias表示**一个MemoryRegion**的**一部分区域**，**MemoryRegion**也可以表示**一个container**，这就表示它**只是其他若干个MemoryRegion的容器**。

**结构体MemoryRegion**是联系**guest物理地址空间**和描述**真实内存的RAMBlocks(宿主机虚拟地址**)之间的桥梁。**每个MemoryRegion结构体**中定义了RAMBlock \***ram\_block**成员指向**其对应的RAMBlock**，而在**RAMBlock**结构体中则定义了struct MemoryRegion \*mr指向**对应的MemoryRegion**。

## 2.3 RAMBlock

```c
// include/exec/ram_addr.h
typedef uint64_t ram_addr_t;

struct RAMBlock {
    struct rcu_head rcu; //该数据结构受rcu机制保护
    struct MemoryRegion *mr;
    uint8_t *host;  //RAMBlock在host上的虚拟内存起始位置
    uint8_t *colo_cache; /* For colo, VM's ram cache */
    ram_addr_t offset;  //在所有的RAMBlock中offset
    ram_addr_t used_length; //已使用长度
    ram_addr_t max_length;  //最大分配内存
    void (*resized)(const char*, uint64_t length, void *host);
    uint32_t flags;
    /* Protected by iothread lock.  */
    char idstr[256];    //RAMBlock的ID
    /* RCU-enabled, writes protected by the ramlist lock */
    QLIST_ENTRY(RAMBlock) next;
    QLIST_HEAD(, RAMBlockNotifier) ramblock_notifiers;
    int fd; //映射文件的描述符
    size_t page_size;
    /* dirty bitmap used during migration */
    unsigned long *bmap;
    unsigned long *unsentmap;
    /* bitmap of already received pages in postcopy */
    unsigned long *receivedmap;
};
```

**RAMBlock结构体**中的

- uint8\_t \***host**指向**动态分配的内存**，用于表示**实际的虚拟机物理内存**，指向**host**上**虚拟内存的起始值**，
- ram\_addr\_t **offset**表示**当前RAMBlock**相对于**RAMList**（描述**host虚拟内存的全局链表**）的**偏移量**。

也就是说**ram\_addr\_t offset**位于一个**全局命名空间**中，可以通过此offset偏移量**定位某个RAMBlock**。

每一个ram\_block还会被连接到全局的'ram\_list'链表上。



Address, MemoryRegion, RAMBlock关系如下图所示。

![](./images/2019-06-10-11-12-09.png)

## 2.4 FlatView

AddressSpace下面**root及其子树**形成了一个**虚拟机的物理地址**，但是在往**kvm进行设置**的时候，需要将其转换为一个**平坦的地址模型**，也就是从0开始的。

这个就用FlatView表示，**一个AddressSpace**对应**一个FlatView**。

```c
// include/exec/memory.h
struct FlatView {
    struct rcu_head rcu;
    unsigned ref;
    FlatRange *ranges;
    unsigned nr;
    unsigned nr_allocated;
    struct AddressSpaceDispatch *dispatch;
    MemoryRegion *root;
};
```

在FlatView中，FlatRange表示按照需要被切分为了几个范围。

## 2.5 MemoryRegionSection

在内存虚拟化中，还有一个重要的结构是MemoryRegionSection，这个结构通过函数section\_from\_flat\_range可由FlatRange转换过来。

```c
// include/exec/memory.h
struct MemoryRegionSection {
    MemoryRegion *mr;
    FlatView *fv;
    hwaddr offset_within_region;
    Int128 size;
    hwaddr offset_within_address_space;
    bool readonly;
    bool nonvolatile;
};
```

MemoryRegionSection表示的是**MemoryRegion的一部分**。这个其实跟FlatRange差不多。

这几个数据结构关系如下：

![](./images/2019-06-10-11-58-47.png)

## 2.6 MemoryListener

为了**监控虚拟机的物理地址访问**，对于**每一个AddressSpace**，会有**一个MemoryListener**与之对应。每当**物理映射（GPA\-\>HVA**)发生改变时，会**回调这些函数**。

所有的MemoryListener都会挂在**全局变量memory\_listeners链表**上。同时，AddressSpace也会有一个链表连接器自己注册的MemoryListener。

```c
// include/exec/memory.h
struct MemoryListener {
    void (*begin)(MemoryListener *listener);
    void (*commit)(MemoryListener *listener);
    void (*region_add)(MemoryListener *listener, MemoryRegionSection *section);
    void (*region_del)(MemoryListener *listener, MemoryRegionSection *section);
    void (*region_nop)(MemoryListener *listener, MemoryRegionSection *section);
    void (*log_start)(MemoryListener *listener, MemoryRegionSection *section,
                      int old, int new);
    void (*log_stop)(MemoryListener *listener, MemoryRegionSection *section,
                     int old, int new);
    void (*log_sync)(MemoryListener *listener, MemoryRegionSection *section);
    void (*log_global_start)(MemoryListener *listener);
    void (*log_global_stop)(MemoryListener *listener);
    void (*eventfd_add)(MemoryListener *listener, MemoryRegionSection *section,
                        bool match_data, uint64_t data, EventNotifier *e);
    void (*eventfd_del)(MemoryListener *listener, MemoryRegionSection *section,
                        bool match_data, uint64_t data, EventNotifier *e);
    void (*coalesced_io_add)(MemoryListener *listener, MemoryRegionSection *section,
                               hwaddr addr, hwaddr len);
    void (*coalesced_io_del)(MemoryListener *listener, MemoryRegionSection *section,
                               hwaddr addr, hwaddr len);
    /* Lower = earlier (during add), later (during del) */
    unsigned priority;
    AddressSpace *address_space;
    QTAILQ_ENTRY(MemoryListener) link;
    QTAILQ_ENTRY(MemoryListener) link_as;
};
```

## 2.7 AddressSpaceDispatch

为了在**虚拟机退出**时，能够顺利根据**虚拟机物理地址**找到**对应的HVA**地址，qemu会有一个AddressSpaceDispatch结构，用来在AddressSpace中进行位置的找寻，继而完成对IO/MMIO地址的访问。

```c
// exec.c
struct AddressSpaceDispatch {
    MemoryRegionSection *mru_section;
    /* This is a multi-level map on the physical address space.
     * The bottom level has pointers to MemoryRegionSections.
     */
    PhysPageEntry phys_map;
    PhysPageMap map;
};
```

这里面有一个**PhysPageMap**，这其实也是**保存**了一个**GPA\-\>HVA**的**一个映射**，通过**多层页表实现**，当**kvm exit退到qemu**之后，通过这个AddressSpaceDispatch里面的map查找对应的MemoryRegionSection，继而找到对应的主机HVA。

这几个结构体的关系如下：

![](./images/2019-06-10-12-16-01.png)

# 3 初始化流程

```c
main()                          // vl.c
    cpu_exec_init_all()         // exec.c
        memory_map_init()       // exec.c
    machine_run_board_init()    // hw/core/machine.c
        machine_class->init()   // pc_init1, hw/i386/pc_piix.c
            pc_cpus_init()      // hw/i386/pc.c
            pc_memory_init()    // hw/i386/pc.c

```

## 3.1 memory\_map\_init: 全局memory space和io space初始化

首先在**main**\-\>**cpu\_exec\_init\_all**\-\>**memory\_map\_init**中对**全局的memory和io**进行初始化，

- **system\_memory**作为**address\_space\_memory**的**根MemoryRegion**，大小涵盖了**整个64位空间的大小**，当然，这是一个**pure contaner**,并**不会分配空间**的，

- **system\_io**作为address\_space\_io的根MemoryRegion，大小为**65536**，也就是平时的**io port空间**。

```c
//exec.c
static MemoryRegion *system_memory;
static MemoryRegion *system_io;

AddressSpace address_space_io;
AddressSpace address_space_memory;
static void memory_map_init(void)
{
    system_memory = g_malloc(sizeof(*system_memory));
    memory_region_init(system_memory, NULL, "system", UINT64_MAX);
    address_space_init(&address_space_memory, system_memory, "memory");

    system_io = g_malloc(sizeof(*system_io));
    memory_region_init_io(system_io, NULL, &unassigned_io_ops, NULL, "io",
                          65536);
    address_space_init(&address_space_io, system_io, "I/O");
}
```

在随后的**cpu初始化**之中，还会**初始化多个AddressSpace**，这些很多都是disabled的，对虚拟机意义不大。

## 3.2 pc\_memory\_init

在初始化虚拟机中, main中由QEMU入参标识QEMU\_OPTION\_m设定了ram\_size参数, 即虚拟机内存的大小, 通过调用machine\_run\_board\_init, 再调用machine\_class\-\>init(machine), 然后一步步传递给pc\_init1函数.

重点在随后的main\-\>pc\_init\_v2\_8\-\>pc\_init1\-\>pc\_memory\_init中，这里面是**分配系统ram**，也是第一次**真正为虚拟机分配物理内存**。

整个过程中，分配内存也不会像MemoryRegion那么频繁，**mr**很多时候是**创建一个alias**，指向**已经存在的mr**的一部分，这也是**alias的作用**，就是把一个mr分割成多个不连续的mr。

### 3.2.1 pc.ram分配

**真正分配空间**的大概有这么几个，**pc.ram**, **pc.bios**, **pc.rom**, 以及**设备的一些ram**, rom等，vga.vram, vga.rom, e1000.rom等。

```c
memory_region_allocate_system_memory(ram, NULL, "pc.ram",
                                    machine->ram_size);
```

分配pc.ram的流程如下：

```
pc_memory_init  // hw/i386/pc.c
memory_region_allocate_system_memory    // numa.c
allocate_system_memory_nonnuma          // numa.c
memory_region_init_ram_shared_nomigrate // memory.c
qemu_ram_alloc      // exec.c
ram_block_add       // exec.c
phys_mem_alloc      // exec.c
qemu_anon_ram_alloc // util/oslib-posix.c
qemu_ram_mmap       // util/mmap-alloc.c
mmap
```

可以看到，qemu通过使用**mmap**创建一个**内存映射**来作为ram。

### 3.2.2 两个mr alias: ram\_below\_4g和ram\_above\_4g

继续**pc\_memory\_init**，函数在创建好了ram并且分配好了空间之后，创建了**两个mr alias**，**ram\_below\_4g**以及**ram\_above\_4g**，这两个mr分别指向**ram的低4g**以及**高4g空间**，这两个alias是挂在**根system\_memory mr下面**的。即高低端内存（也不一定是32bit机器）

```c
ram_below_4g = g_malloc(sizeof(*ram_below_4g));
memory_region_init_alias(ram_below_4g, NULL, "ram-below-4g", ram,
                            0, pcms->below_4g_mem_size);
memory_region_add_subregion(system_memory, 0, ram_below_4g);
e820_add_entry(0, pcms->below_4g_mem_size, E820_RAM);
if (pcms->above_4g_mem_size > 0) {
    ram_above_4g = g_malloc(sizeof(*ram_above_4g));
    memory_region_init_alias(ram_above_4g, NULL, "ram-above-4g", ram,
                            pcms->below_4g_mem_size,
                            pcms->above_4g_mem_size);
    memory_region_add_subregion(system_memory, 0x100000000ULL,
                                ram_above_4g);
    e820_add_entry(0x100000000ULL, pcms->above_4g_mem_size, E820_RAM);
}
```

以后的情形类似，创建根mr，创建AddressSpace，然后在根mr下面加subregion。

# 4 内存的提交

当我们每一次**更改上层的内存布局**之后，都需要**通知到kvm**。这个过程是通过一系列的**MemoryListener来实现**的。

首先系统有一个**全局的memory\_listeners**，上面挂上了**所有的MemoryListener**，在address\_space\_init\-\>address\_space\_init\_dispatch\-\>memory\_listener\_register这个过程中完成**MemoryListener的注册**。

每个address\_space都对应一个memory\_listener, 所以在创建address\_space即会注册memory\_listener.

