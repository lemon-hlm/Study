
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Qemu内存分布](#1-qemu内存分布)
* [2 内存初始化](#2-内存初始化)
* [3 内存分配](#3-内存分配)

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

AddressSpace设置了一段内存，其主要信息存储在root成员 中，root成员是个MemoryRegion结构，主要存储内存区的结构。在Qemu中最主要的两个AddressSpace是 address\_space_memory和address_space_io，分别对应的MemoryRegion变量是system_memory和 system\_io。

Qemu的**主函数**是**vl.c**中的main函数，其中调用了**configure\_accelerator**()，是KVM初始化的配置部分。

configure\_accelerator中首先根据命令行输入的参数找到对应的accelerator，这里是KVM。之后调用accel\_list\[i].init()，即kvm\_init()。

在kvm_init()函数中主要做如下几件事情：

1. s\-\>fd = qemu\_open("/dev/kvm", O\_RDWR)，**打开kvm控制的总设备文件/dev/kvm**
2. s\-\>vmfd = kvm\_ioctl(s, KVM\_CREATE\_VM, 0)，调用**创建虚拟机的API**
3. kvm\_check\_extension，检查各种extension，并设置对应的features
4. ret = kvm\_arch\_init(s)，做一些**体系结构相关的初始化**，如**msr**、**identity map**、**mmu pages number**等等
5. kvm\_irqchip\_create，调用kvm\_vm\_ioctl(s, KVM\_CREATE\_IRQCHIP)**在KVM中虚拟IRQ芯片**
6. **memory\_listener\_register**，该函数是**初始化内存**的主要函数
7. **memory\_listener\_register调用了两次**，分别注册了 **kvm\_memory\_listener**和**kvm\_io\_listener**，即**通用的内存**和**MMIO**是**分开管理**的。

以**通用的内存注册**为例，函数首先在**全局的memory\_listener链表**中添加了**kvm\_memory\_listener**，之后调用**listener\_add\_address\_space**分别将**该listener**添加到**address\_space\_memory**和**address\_space\_io**中, address\_space\_io是虚机的io地址空间（**设备的io port**就分布在这个地址空间里）。

8. 然后**调用listener的region\_add**（即 kvm_region_add()），该函数最终调用了kvm\_set\_user\_memory\_region()，其中调用 kvm\_vm\_ioctl(s, **KVM\_SET\_USER\_MEMORY\_REGION**, &mem)，该调用是最终**将内存区域注册到kvm**中的函数。
9. 之后在vl.c的main函数中调用了cpu\_exec\_init\_all() \=\> memory\_map\_init()，设置**system\_memory**和**system\_io**。

至此**初始化好**了所有Qemu中需要维护的**相关的内存结构**，并完成了在KVM中的注册。下面需要初始化KVM中的MMU支持。

ram\_size内存大小从内存被读取到ram\_size中，在vl.c的main中调用machine\-\>init()来初始化，**machine**是命令行**指定的机器类型**，默认的init是pc\_init\_pci

```c
pc_memory_init
memory_region_allocate_system_memory
memory_region_add_subregion
memory_region_add_subregion_common
memory_region_update_container_subregions
memory_region_transaction_commit
address_space_update_topology
generate_memory_topology
render_memory_region
flatview_insert
```

# 3 内存分配

**内存的分配**实现函数为 ram\_addr\_t **qemu\_ram\_alloc**(ram\_addr\_t size, MemoryRegion \*mr), 输出为**该次分配的内存**在**所有分配内存**中的**顺序偏移**(即下图中的红色数字). 

该函数最终调用phys\_mem\_alloc分配内存, 并将所分配的全部内存块, 串在一个ram\_blocks开头的链表中, 如下示意:

![](./images/2019-06-16-14-06-50.png)



参考

https://blog.51cto.com/zybcloud/2149626