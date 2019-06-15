
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Qemu内存分布](#1-qemu内存分布)
* [2 内存初始化](#2-内存初始化)

<!-- /code_chunk_output -->

# 1 Qemu内存分布

![](./images/2019-06-15-22-08-00.png)

# 2 内存初始化

Qemu中的内存模型，简单来说就是Qemu**申请用户态内存**并进行管理，并将该部分申请的内存**注册**到**对应的加速器（如KVM）中**。

这样的模型有如下好处：

1. **策略与机制分离**。**加速的机制**由**KVM**负责，而**如何调用**加速的机制由**Qemu负责**

2. 可以由**Qemu**设置**多种内存模型**，如**UMA**、**NUMA**等等

3. 方便Qemu**对特殊内存的管理**（如MMIO）

4. 内存的**分配**、**回收**、**换出**等都可以采用Linux原有的机制，**不需要**为KVM**单独开发**。

5. **兼容其他加速器模型**（或者**无加速器**，单纯使用**Qemu做模拟**）

Qemu需要做的有两方面工作：

- **向KVM注册用户态内存空间**，
- 申请**用户态内存空间**。

Qemu主要通过如下结构来维护内存：

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

"memory"的root是static MemoryRegion system_memory;
使用**链表address\_spaces**保存**虚拟机的内存**，该链表保存AddressSpace address\_space\_io和AddressSpace address\_space\_memory等信息

```c
// memory.c
void address_space_init(AddressSpace *as, MemoryRegion *root, const char *name)
{
    memory_region_ref(root);
    as->root = root;
    as->current_map = NULL;
    as->ioeventfd_nb = 0;
    as->ioeventfds = NULL;
    QTAILQ_INIT(&as->listeners);
    QTAILQ_INSERT_TAIL(&address_spaces, as, address_spaces_link);
    as->name = g_strdup(name ? name : "anonymous");
    address_space_update_topology(as);
    address_space_update_ioeventfds(as);
}
```

AddressSpace设置了一段内存，其主要信息存储在root成员 中，root成员是个MemoryRegion结构，主要存储内存区的结构。在Qemu中最主要的两个AddressSpace是 address_space_memory和address_space_io，分别对应的MemoryRegion变量是system_memory和 system_io。

Qemu的主函数是vl.c中的main函数，其中调用了configure_accelerator()，是KVM初始化的配置部分。

configure_accelerator中首先根据命令行输入的参数找到对应的accelerator，这里是KVM。之后调用accel_list[i].init()，即kvm_init()。

在kvm_init()函数中主要做如下几件事情：

参考

https://blog.51cto.com/zybcloud/2149626