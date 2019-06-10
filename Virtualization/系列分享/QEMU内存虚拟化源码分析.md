
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [0 内存虚拟化](#0-内存虚拟化)
* [1 概述](#1-概述)
* [2 相关数据结构](#2-相关数据结构)

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

首先，qemu中用**AddressSpace**用来表示**CPU/设备**看到的**内存**，一个AddressSpace下面包含**多个MemoryRegion**，这些MemoryRegion结构通过树连接起来，树的根是AddressSpace的root域。

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
    // 分配的实际内存
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

MemoryRegion有**多种类型**，可以表示一段**ram**、**rom**、**MMIO**、**alias**，alias表示**一个MemoryRegion**的**一部分区域**，**MemoryRegion**也可以表示**一个container**，这就表示它**只是其他若干个MemoryRegion的容器**。在MemoryRegion中，'ram\_block'表示的是**分配的实际内存**。