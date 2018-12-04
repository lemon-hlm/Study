[TOC]

- 1 进程ID概述
    - 1.1 进程ID类型
    - 1.2 PID命名空间
        - 1.2.1 pid命名空间概述
        - 1.2.2 局部ID和全局ID
        - 1.2.3 PID命名空间数据结构pid\_namespace
- 2 pid结构描述
    - 2.1 pid与upid
    - 2.2 pidmap用于分配pid的位图
    - 2.3 pid\_link哈希表存储
    - 2.4 task\_struct中的进程ID相关描述符信息
- 3 内核是如何设计task\_struct中进程ID相关数据结构的
    - 3.1 一个PID对应一个task时的task\_struct设计
    - 3.2 如何快速地根据局部ID、命名空间、ID类型找到对应进程的task\_struct
    - 3.3 如何快速地给新进程在可见的命名空间内分配一个唯一的PID
    - 3.4 带进程ID类型的task\_struct设计
    - 3.5 进一步增加进程PID命名空间的task\_struct设计
- 4 进程ID管理函数
    - 4.1 pid号到struct pid实体
    - 4.2 获得局部ID
    - 4.3 根据PID查找进程task\_struct
    - 4.4 生成唯一的PID

Linux 内核使用 task\_struct 数据结构来关联所有与进程有关的数据和结构，Linux 内核所有涉及到进程和程序的所有算法都是围绕该数据结构建立的，是内核中最重要的数据结构之一。

该数据结构在内核文件[include/linux/sched.h](http://lxr.free-electrons.com/source/include/linux/sched.h#L1389)中定义，在目前最新的Linux-4.5（截至目前的日期为2016-05-11）的内核中，该数据结构足足有 380 行之多，在这里我不可能逐项去描述其表示的含义，本篇文章只关注该数据结构如何来组织和管理进程ID的。

# 1 进程ID概述

## 1.1 进程ID类型

要想了解内核如何来组织和管理进程ID，先要知道**进程ID的类型**：

内核中进程ID的类型用[pid\_type](http://lxr.free-electrons.com/source/include/linux/pid.h#L6)来描述,它被定义在[include/linux/pid.h](http://lxr.free-electrons.com/source/include/linux/pid.h)中

```c
[include/linux/pid.h]
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};
```

之所以**不包括线程组ID**，是因为内核中已经有指向到**线程组**的**task\_struct**指针**group\_leader**，线程组ID无非就是**group\_leader的PID**。

- **PID**   内核唯一区分**每个进程的标识**

pid是 Linux 中在其**命名空间**中**唯一标识进程**而分配给它的一个号码，称做**进程ID号，简称PID**。在使用 fork 或 clone 系统调用时产生的进程均会由内核分配一个新的唯一的PID值

这个pid用于内核唯一的区分每个进程

>注意它并不是我们用户空间通过getpid()所获取到的那个进程号，至于原因么，接着往下看

- **TGID**  **线程组（轻量级进程组**）的ID标识

在一个进程中，如果以**CLONE\_THREAD标志**来调用clone建立的进程就是**该进程的一个线程**（即**轻量级进程**，Linux其实**没有严格的线程概念**），它们处于一个线程组，

该**线程组的所有线程的ID叫做TGID**。处于相同的线程组中的所有进程都有相同的TGID，但是由于他们是不同的进程，因此其pid各不相同；**线程组组长（也叫主线程**）的TGID与其PID相同；一个**进程没有使用线程**，则其TGID与PID也**相同**。

- **PGID**    **进程组**的ID标识

另外，**独立的进程可以组成进程组**（使用**setpgrp系统调用**），进程组可以简化向所有组内进程发送信号的操作

例如**用管道连接的进程处在同一进程组内**。进程组ID叫做PGID，进程组内的所有进程都有相同的PGID，等于**该组组长的PID**。

- **SID**     **会话组**的ID标识

**几个进程组**可以合并成**一个会话组**（使用**setsid系统调用**），可以用于终端程序设计。会话组中所有进程都有相同的SID,保存在**task\_struct的session**成员中

## 1.2 PID命名空间

### 1.2.1 pid命名空间概述

**命名空间**是为**操作系统层面的虚拟化机制**提供支撑，目前实现的有**六种不同的命名空间**，分别为**mount命名空间、UTS命名空间、IPC命名空间、用户命名空间、PID命名空间、网络命名空间**。命名空间简单来说提供的是对**全局资源的一种抽象**，将资源放到不同的容器中（不同的命名空间），各容器彼此隔离。

命名空间**有的**还有**层次关系**，如**PID命名空间**

PID命名空间的层次关系图:

![命名空间的层次关系图](./images/namespace-level.jpg)

在上图有四个命名空间，一个父命名空间衍生了两个子命名空间，其中的一个子命名空间又衍生了一个子命名空间。以PID命名空间为例，由于各个命名空间彼此隔离，所以每个命名空间都可以有 PID 号为 1 的进程；但又由于命名空间的层次性，父命名空间是知道子命名空间的存在，因此子命名空间要映射到父命名空间中去，因此上图中 level 1 中两个子命名空间的六个进程分别映射到其父命名空间的PID号5\~10。

### 1.2.2 局部ID和全局ID

命名空间增加了**PID管理的复杂性**。

回想一下，PID命名空间按**层次组织**。在建立一个新的命名空间时，**子命名空间中的所有PID**对**父命名空间都是可见**的，但子命名空间无法看到父命名空间的PID。但这意味着某些进程具有多个PID，凡可以看到该进程的命名空间，都会为其分配一个PID。这必须反映在数据结构中。我们必须区分**局部ID**和**全局ID**

- **全局ID**    在内核本身和**初始命名空间中唯一的ID(初始命名空间中！！！**)，在系统启动期间开始的init进程即属于该初始命名空间。系统中**每个进程**都对应了该命名空间的一个**PID**，叫**全局ID**，保证在整个系统中唯一。

- **局部ID**    对于属于**某个特定的命名空间**，它在其命名空间内分配的ID为局部ID，该ID也可以出现在其他的命名空间中。

**全局PID和全局TGID**直接保存在task\_struct中，分别是task\_struct的**pid和tgid成员**：

```c
<sched.h> 
struct task_struct
{
    pid_t pid;  
    pid_t tgid; 
}
```
 
两项都是**pid\_t类型**，该**类型定义为\_\_kernel\_pid\_t**，后者由**各个体系结构分别定义**。通常定义为int，即可以同时使用232个不同的ID。

**会话session和进程group组ID**不是直接包含在task\_struct本身中，但保存在**用于信号处理的结构**中。

- **task\_struct->signal->\_\_session**表示**全局SID**，

- 而**全局PGID**则保存在**task\_struct->signal->\_\_pgrp**。

辅助函数**set\_task\_session**和**set\_task\_pgrp**可用于修改这些值。

除了这两个字段之外，内核还需要找一个办法来管理所有**命名空间内部的局部量**，以及其他ID（如TID和SID）。这需要几个相互连接的数据结构，以及许多辅助函数，并将在下文讨论。

下文我将使用ID指代提到的任何进程ID。在必要的情况下，我会明确地说明ID类型（例如，TGID，即线程组ID）。

一个**小型的子系统**称之为**PID分配器（pid allocator**）用于加速新ID的分配。此外，内核需要提供辅助函数，以实现通过ID及其类型查找进程的task\_struct的功能，以及将ID的内核表示形式和用户空间可见的数值进行转换的功能。

### 1.2.3 PID命名空间数据结构pid\_namespace

在介绍表示ID本身所需的数据结构之前，我需要讨论**PID命名空间的表示方式**。我们所需查看的代码如下所示：

[pid\_namespace](http://lxr.free-electrons.com/source/include/linux/pid_namespace.h#L24)的定义在[include/linux/pid\_namespace.h](http://lxr.free-electrons.com/source/include/linux/pid_namespace.h#L24)中

命名空间的结构如下

```c
struct pid_namespace
{  
    struct kref kref;  
    struct pidmap pidmap[PIDMAP_ENTRIES];  
    int last_pid;  
    struct task_struct *child_reaper;  
    struct kmem_cache *pid_cachep;  
    unsigned int level;  
    struct pid_namespace *parent;
}; 
```

>我们这里只关心其中的child\_reaper，level和parent这三个字段

| 字段| 描述 | 
| ------------- |:-------------|
| kref | 表示**指向pid\_namespace的个数** |
| pidmap | pidmap结构体表示**分配pid的位图**。当需要分配一个新的pid时只需查找位图，找到**bit为0的位置并置1**，然后**更新统计数据域**（nr\_free) |
| last\_pid | 用于pidmap的分配。指向最后一个分配的pid的位置。(不是特别确定）|
| child\_reaper | 指向的是**当前命名空间的init进程**，每个命名空间都有一个作用相当于全局init进程的进程 |
| pid\_cachep | 域指向**分配pid的slab的地址**。|
| level | 代表**当前命名空间的等级**，**初始命名空间的level为0**，它的子命名空间level为1，依次递增，而且子命名空间对父命名空间是可见的。从**给定的level**设置，内核即可推断**进程会关联到多少个ID**。|
| parent | 指向**父命名空间的指针** |

![PID命名空间.png](./images/pid-namespace.png)

实际上PID分配器也需要依靠该结构的某些部分来连续生成唯一ID，但我们目前对此无需关注。我们上述代码中给出的下列成员更感兴趣。

**每个PID命名空间都具有一个进程**，其发挥的作用**相当于全局的init进程**。init的一个目的是**对孤儿进程调用wait4**，命名空间**局部的init变体**也必须完成该工作。

# 2 pid结构描述

## 2.1 pid与upid

**PID的管理**围绕两个数据结构展开：

- struct pid是**内核对PID的内部表示**

- struct upid则表示**特定的命名空间中可见**的信息。

两个结构的定义在[include/linux/pid.h](include/linux/pid.h)中

```c
struct upid
{  
    /* Try to keep pid_chain in the same cacheline as nr for find_vpid */
    int nr;  
    struct pid_namespace *ns;  
    struct hlist_node pid_chain;  
};  
```

struct upid是一个**特定namespace**里面的进程的信息,包含该namespace里面进程具体ID号,namespace指针,哈希列表指针.

| 字段| 描述 | 
| ------------- |:-------------|
| nr | 表示**在该命名空间所分配的进程ID具体的值** |
| ns | 指向**命名空间的指针** |
| pid\_chain | 指向**PID哈希列表的指针**，用于**关联对应的PID** |

**所有的upid实例**都保存在一个**散列表**中，稍后我们会看到该结构。

```c
[include/linux/pid.h]
struct pid  
{  
    atomic_t count;  
    /* 使用该pid的进程的列表  */
    struct hlist_head tasks[PIDTYPE_MAX];  
    int level;  
    struct upid numbers[1];  
};
```

srtuct pid是**局部ID类**,对应一个

| 字段| 描述 | 
| ------------- |:-------------|
| count | 是指**使用该PID的task的数目**；|
| level | 表示可以看到**该PID的命名空间的数目**，也就是包含该进程的命名空间的深度 |
| tasks[PIDTYPE\_MAX] | 是一个**数组**，每个**数组项**都是一个**散列表头**,分别对应以下三种类型
| numbers[1] | 一个**upid**的**实例数组**，每个数组项代表一个**命名空间**，用来表示**一个PID**可以属于**不同的命名空间**，该元素放在末尾，**可以向数组添加附加的项**。|

tasks是一个数组，**每个数组项**都是一个**散列表头**，对应于一个ID类型, PIDTYPE\_PID,PIDTYPE\_PGID,PIDTYPE\_SID（PIDTYPE\_MAX表示**ID类型的数目**）这样做是必要的，因为**一个ID可能用于几个进程(task\_struct)！！！**。所有**共享同一ID**的**task\_struct实例**，都**通过该列表连接起来(这个列表就是使用这个pid的进程<task\_struct>的列表！！！**)。这个枚举常量PIDTYPE\_MAX，正好是pid\_type类型的数目，这里linux内核使用了一个小技巧来由编译器来自动生成id类型的数目

此外，还有两个结构我们需要说明，就是pidmap和pid\_link

- pidmap当需要分配一个新的pid时查找可使用pid的位图

- 而pid\_link则是pid的哈希表存储结构

## 2.2 pidmap用于分配pid的位图

```c
struct pidmap
{  
	atomic_t nr_free;  
	void *page; 
};
```

| 字段| 描述 | 
| ------------- |:-------------:|
| nr\_free | 表示**还能分配的pid的数量** |
| page | 指向的是**存放pid的物理页** |

pidmap[PIDMAP\_ENTRIES]域表示该pid\_namespace下pid已分配情况

## 2.3 pid\_link哈希表存储

task\_struct中的pids[PIDTYPE\_MAX]指向了**和该task\_struct相关的pid结构体**。

pid\_link的定义如下

```c
struct pid_link  
{  
    struct hlist_node node;  
    struct pid *pid;  
};
```

## 2.4 task\_struct中的进程ID相关描述符信息

```c
struct task_struct  
{  
    //...  
    pid_t pid;  
    pid_t tgid;  
    struct task_struct *group_leader;  
    struct pid_link pids[PIDTYPE_MAX];  
    struct nsproxy *nsproxy;  
    //...  
};
```
| 字段| 描述 | 
| ------------- |:-------------|
| pid | 指该进程的**进程描述符**。在**fork函数**中对其进行**赋值**的 |
| tgid | 指该进程的**线程描述符**。在linux内核中对线程并没有做特殊的处理，还是由task\_struct来管理。所以从内核的角度看， **用户态的线程本质上还是一个进程**。对于**同一个进程**（用户态角度）中不同的线程其tgid是相同的，但是pid各不相同。 **主线程即group\_leader**（主线程会创建其他所有的子线程）。如果是单线程进程（用户态角度），它的pid等于tgid。|
| group\_leader | 除了在**多线程的模式下指向主线程**，还有一个用处，当一些**进程组成一个群组**时（**PIDTYPE\_PGID**)， 该域指向该**群组的leader** |
| pids | pids[0]是PIDTYPE\_PID类型的,指向自己的PID结构, 其余指向了**相应群组的leader的PID结构**,也就是组长的PID结构 |
| nsproxy | 指针指向**namespace相关的域**，通过nsproxy域可以知道**该task\_struct属于哪个pid\_namespace** |

对于用户态程序来说，调用**getpid**()函数其实返回的是**tgid**，因此线程组中的进程id应该是是一致的，但是他们pid不一致，这也是内核区分他们的标识

1. **多个task\_struct**可以共用**一个PID**

2. **一个PID**可以属于**不同的命名空间**

3.	当需要**分配一个新的pid**时候，只需要查找**pidmap位图**即可

那么最终，linux下进程命名空间和进程的关系结构如下：

![进程命名空间和进程的关系结构](./images/pidnamespace-and-process.png)

可以看到，**多个task\_struct**指向**一个PID**，同时PID的hash数组里安装不同的类型对task进行散列，并且一个PID会属于多个命名空间。

# 3 内核是如何设计task\_struct中进程ID相关数据结构的

>本部内容较多的采用了[Linux 内核进程管理之进程ID](http://www.cnblogs.com/hazir/p/linux_kernel_pid.html)

Linux 内核在设计管理ID的数据结构时，要充分考虑以下因素：

1. 如何快速地根据**进程的task\_struct**、**ID类型**、**命名空间**找到**局部ID**

2. 如何快速地根据**局部ID**、**命名空间**、**ID类型**找到对应**进程的task\_struct**

3. 如何快速地给**新进程**在可见的**命名空间内**分配一个**唯一的PID**

如果将所有因素(主要是**进程的task\_struct,ID类型,命名空间,局部ID,唯一ID**这5个因素)考虑到一起，将会很复杂，下面将会**由简到繁设计该结构**。

## 3.1 一个PID对应一个task时的task\_struct设计

一个PID对应一个task\_struct如果先**不考虑进程之间的关系**，**不考虑命名空间**，仅仅是**一个PID号对应一个task\_struct(一对一情况)**，那么我们可以设计这样的数据结构

```c
struct task_struct
{
    //...
    struct pid_link pids;   
    //...
};

struct pid_link
{
    struct hlist_node node;
    struct pid *pid;
};

struct pid
{
    struct hlist_head tasks;     //指回 pid_link 的 node
    int nr;                      //PID
    struct hlist_node pid_chain; //pid hash 散列表结点
};
```
每个进程的task\_struct结构体中有**一个指向pid结构体的指针(一对一情况下1个就够了！！！**)，pid结构体包含了PID号。

结构示意图如图

![一个task_struct对应一个PID](./images/per-task_struct-per-pid.png)

## 3.2 如何快速地根据局部ID、命名空间、ID类型找到对应进程的task\_struct

图中还有两个结构上面未提及：

- pid\_hash[]

这是一个**hash表**的结构，**根据pid的nr值哈希到其某个表项**，若有**多个pid**结构对应到**同一个表项**，这里**解决冲突**使用的是**散列表法**。

这样，就能解决开始提出的第2个问题了，根据**PID值**怎样快速地找到**task\_struct**结构体：

1. 首先通过PID计算pid挂接到**哈希表pid\_hash[]的表项**

2. **遍历该表项**,找到**pid结构体(实际是upid结构体**)中**nr值**与**PID值相同**的那个pid

3. 再通过该pid结构体的**tasks指针**找到**node**(一对一情况下,直接将task指回pid\_link的node就行)

4. 最后根据内核的container\_of机制就能找到task\_struct结构体

## 3.3 如何快速地给新进程在可见的命名空间内分配一个唯一的PID

- pid\_map

这是一个位图，用来**唯一分配PID值的结构**，图中灰色表示已经分配过的值，在新建一个进程时，只需在其中找到一个为分配过的值赋给pid结构体的nr，再将pid\_map中该值设为已分配标志。这也就解决了上面的**第3个问题——如何快速地分配一个全局的PID**

至于上面的**第1个问题*就更加简单，已知task\_struct结构体，根据其pid\_link的pid指针找到pid结构体，取出其nr即为PID号。

## 3.4 带进程ID类型的task\_struct设计

如果考虑进程之间有复杂的关系，如**线程组、进程组、会话组，这些组均有组ID**，分别为TGID、PGID、SID，所以原来的task\_struct中**pid\_link指向一个pid结构体需要增加几项**，用来指向到**其组长的pid结构体**，相应的**struct pid**原本只需要指回其PID所属进程的task\_struct，现在要增加几项，用来链接那些以**该pid为组长的所有进程组内进程**。数据结构如下：

>定义在http://lxr.free-electrons.com/source/include/linux/sched.h#L1389

```c
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};

struct task_struct
{
    pid_t pid; //PID
    pid_t tgid; //thread group id
    struct task_struct *group_leader; // threadgroup leader
    struct pid_link pids[PIDTYPE_MAX];  
    struct nsproxy *nsproxy;
};

struct pid_link
{
    struct hlist_node node;
    struct pid *pid;
};

struct pid
{
    struct hlist_head tasks[PIDTYPE_MAX];
    int nr; //PID
    struct hlist_node pid_chain; // pid hash 散列表结点
};

struct pid_link
{
    struct hlist_node node;
    struct pid *pid;
};
```

上面ID的类型PIDTYPE\_MAX表示**ID类型数目**。之所以**不包括线程组ID**，是因为内核中已经有指向到**线程组**的**task\_struct**指针**group\_leader**，线程组ID无非就是**group\_leader的PID**。

假如现在有三个进程A、B、C为同一个**进程组**，进程**组长为A**，这样的结构示意图如图

增加ID类型的结构:

![增加ID类型的结构](./images/task_struct-with-pidtype.png)

关于上图有几点需要说明：

- 图中省去了pid\_hash以及pid\_map结构，因为第一种情况类似；

- 三个进程A,B,C.所以有三个task\_struct结构体. **task\_struct A的pids[0]指向pid**,因为**pids[0]就是PID的类型0对应的是PIDTYPE\_PID,表明pids[0]指向的pid就是task\_struct A的PID,而不是进程组或者会话组的组长的pid结构！！！**所以图中的结构体pid是进程A的pid

- 进程B和C的**进程组组长为A(进程组组长！！！**)，那么**进程B和进程C结构task\_struct**中的pids[**PIDTYPE\_PGID(进程组)**]的pid指针指向**进程A的pid结构体**,如图；进程B和进程C的pids[PIDTYPE\_PID]会指向自己的pid结构体,图中没有体现而已

- 进程A是进程B和C的组长，**进程A的pid结构体**的**tasks[PIDTYPE\_PGID(进程组**)]（tasks[1]）是一个**散列表的头**，它将所有**以该pid为组长的进程链接起来**,图中的结构体pid是进程A的PID,而**A是组长**,B和C是**进程组**的组员,所以将A的pid中tasks[PIDTYPE\_PGID]指向进程B的task\_struct中的pid\_link[PIDTYPE\_PGID]中的node.

再次回顾本节的三个基本问题，在此结构上也很好去实现。

## 3.5 进一步增加进程PID命名空间的task\_struct设计

若在第二种情形下再**增加PID命名空间**

**一个进程**就可能有**多个PID**(**意味着一个task\_struct会对应多个upid???**)值了，因为在**每一个可见的命名空间**内都会**分配一个PID**，这样就需要改变pid的结构了，如下：

```c
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};

struct task_struct
{
    pid_t pid; //PID
    pid_t tgid; //thread group id
    struct task_struct *group_leader; // threadgroup leader
    struct pid_link pids[PIDTYPE_MAX];  
    struct nsproxy *nsproxy;
};

struct pid_link
{
    struct hlist_node node;
    struct pid *pid;
};

struct pid
{
    unsigned int level;
    /* 使用该pid的进程的列表， lists of tasks that use this pid  */
    struct hlist_head tasks[PIDTYPE_MAX];
    struct upid numbers[1];
};

struct upid
{
    int nr;
    struct pid_namespace *ns;
    struct hlist_node pid_chain;
};
```

在pid结构体中增加了一个表示该**进程所处的命名空间的层次level**，以及一个**可扩展的upid结构体**。

对于struct upid，nr表示在**该命名空间**所分配的**进程的ID**，ns指向是该ID所属的命名空间，**pid\_chain 表示在该命名空间的散列表**。

- 进程的结构体是task\_struct,**一个进程对应一个task\_struct结构体(一对一**).**一个进程**会有**PIDTYPE\_MAX个(3个)pid\_link结构体(一对多**),这**三个结构体中的pid**分别指向①该进程对应的**进程本身(PIDTYPE\_PID**)的真实的pid结构体();②该进程的**进程组(PIDTYPE\_PGID)的组长本身**的pid结构体;③该进程的**会话组(PIDTYPE\_SID)的组长**本身的pid结构体,所以**一个真实的进程只会有一个自身真实的pid结构体**
- 这三个pid\_link结构体里面有个哈希节点node,因为进程组、会话组等的存在,这个**node用来链接同一个组的进程task\_struct**,指向的是task\_struct中的pid\_link的node
- pid结构体(不是一个ID号)代表**一个真实的进程(某个组的组长的pid也是这个结构体,因为组长也是真实的进程,也就有相应的真实的pid结构体,而组长身份是通过task\_struct引的**),所以里面会有①**该进程真实所处命名空间的level**;②**PIDTYPE\_MAX个(3个)散列表头**,tasks[PIDTYPE\_PID]指向自身进程(因为PIDTYPE\_PID是PID类型),如果该进程是进程组组长,那么tasks[PIDTYPE\_PGID]就是这个散列表的表头,指向下一个进程的相应组变量pids[PIDTYPE\_PGID]的node,如果该进程是会话组组长,那么tasks[PIDTYPE\_SID]就是这个散列表的表头,指向下一个进程的相应组变量pids[PIDTYPE\_SID]的node;③由于一个进程可能会呈现在多个pid命名空间,所以有该进程在其他命名空间中的信息结构体upid的数组,每个数组项代表一个
- 结构体upid的数组number[1],**数组项个数取决于该进程pid的level值**,**每个数组项代表一个命名空间**,这个就是用来一个PID可以属于不同的命名空间,nr值表示该进程在该命名空间的pid值,ns指向该信息所在的命名空间,pid\_chain属于哈希表的节点.系统有一个pid\_hash[],通过pid在某个命名空间的nr值哈希到某个表项,如果多个nr值哈希到同一个表项,将其加入链表,这个节点就是upid的pid\_chain

举例来说，在**level 2**的**某个命名空间**上新建了一个进程，分配给它的**pid为45**，映射到**level 1的命名空间**，分配给它的**pid为134**；再映射到**level 0的命名空间**，分配给它的**pid为289**，对于这样的例子，如图所示为其表示：

增加PID命名空间之后的结构图:

![增加PID命名空间之后的结构图](./images/task_struct-with-namespace.png)

图中关于**如何分配唯一的PID没有画出**，但也是比较简单，与前面两种情形不同的是，这里分配唯一的PID是有命名空间的容器的，在**PID命名空间内必须唯一**，但各个命名空间之间不需要唯一。

至此，已经与 Linux 内核中数据结构相差不多了。

# 4 进程ID管理函数

有了上面的复杂的数据结构，再加上散列表等数据结构的操作，就可以写出我们前面所提到的三个问题的函数了

## 4.1 pid号到struct pid实体

很多时候在写内核模块的时候，需要通过**进程的pid找到对应进程的task\_struct**，其中首先就需要通过**进程的pid找到进程的struct pid**，然后再**通过struct pid找到进程的task\_struct**

我知道的实现函数有三个。

```c
//通过pid值找到进程的struct pid实体
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
struct pid *find_vpid(int nr)
struct pid *find_get_pid(pid_t nr)
```

find\_pid\_ns获得pid实体的实现原理，**主要使用哈希查找**。内核使用**哈希表组织struct pid**，每创建一个**新进程**，给进程的struct pid都会**插入到哈希表**中，这时候就需要使用进程的**进程pid**和pid命名空间ns在哈希表中将相对应的struct pid索引出来，现在可以看下find\_pid\_ns的传入参数，也是通过nr和ns找到struct pid。

根据**局部PID**以及**命名空间**计算在**pid\_hash数组中的索引**，然后**遍历散列表**找到**所要的upid**，再根据内核的 container\_of 机制找到 pid 实例。

代码如下：

```c
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
        struct hlist_node *elem;
        struct upid *pnr; 
        //遍历散列表
        hlist_for_each_entry_rcu(pnr, elem,
                        &pid_hash[pid_hashfn(nr, ns)], pid_chain)  //pid_hashfn() 获得hash的索引
                if (pnr->nr == nr && pnr->ns == ns) ////比较 nr 与 ns 是否都相同
                        return container_of(pnr, struct pid,  //根据container_of机制取得pid 实体
                                        numbers[ns->level]);

        return NULL;
}
```

而另外两个函数则是对其进行进一步的封装，如下

```c
struct pid *find_vpid(int nr)
{
        return find_pid_ns(nr, current->nsproxy->pid_ns);
}
struct pid *find_get_pid(pid_t nr)
{ 
        struct pid *pid; 

        rcu_read_lock();
        pid = get_pid(find_vpid(nr)); 
        rcu_read_unlock();

        return pid; 
}
```

三者之间的调用关系如下

![调用关系](./images/pid-to-struct_pid.png)

由图可以看出，find_pid_ns是最终的实现，find_vpid是使用find_pid_ns
实现的，find_get_pid又是由find_vpid实现的。

由原代码可以看出find_vpid和find_pid_ns是一样的，而find_get_pid和find_vpid有一点差异，就是使用find_get_pid将返回的struct pid中的字段count加1，而find_vpid没有加1。

## 4.2 获得局部ID

根据进程的 task\_struct、ID类型、命名空间，可以很容易获得其在命名空间内的局部ID

获得与task\_struct 关联的pid结构体。辅助函数有 task\_pid、task\_tgid、task\_pgrp和task\_session，分别用来获取不同类型的ID的pid 实例，如获取 PID 的实例：

```c
static inline struct pid *task_pid(struct task_struct *task)
{
	return task->pids[PIDTYPE_PID].pid;
}
```

获取线程组的ID，前面也说过，TGID不过是线程组组长的PID而已，所以：
```c
static inline struct pid *task_tgid(struct task_struct *task)
{
	return task->group_leader->pids[PIDTYPE_PID].pid;
}
```

而获得PGID和SID，首先需要找到该线程组组长的task\_struct，再获得其相应的 pid：

```c
static inline struct pid *task_pgrp(struct task_struct *task)
{
	return task->group_leader->pids[PIDTYPE_PGID].pid;
}

static inline struct pid *task_session(struct task_struct *task)
{
	return task->group_leader->pids[PIDTYPE_SID].pid;
}
```

获得 pid 实例之后，再根据 pid 中的numbers 数组中 uid 信息，获得局部PID。

```c
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
	struct upid *upid;
	pid_t nr = 0;
	if (pid && ns->level <= pid->level)
    {
		upid = &pid->numbers[ns->level];
		if (upid->ns == ns)
			nr = upid->nr;
	}
	return nr;
}
```

这里值得注意的是，由于PID命名空间的层次性，父命名空间能看到子命名空间的内容，反之则不能，因此，函数中需要确保当前命名空间的level 小于等于产生局部PID的命名空间的level。

除了这个函数之外，内核还封装了其他函数用来从 pid 实例获得 PID 值，如 pid\_nr、pid\_vnr 等。在此不介绍了。

结合这两步，内核提供了更进一步的封装，提供以下函数：

```c
pid_t task_pid_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
pid_t task_tgid_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
pid_t task_pigd_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
pid_t task_session_nr_ns(struct task_struct *tsk, struct pid_namespace *ns);
```
从函数名上就能推断函数的功能，其实不外于封装了上面的两步。

## 4.3 根据PID查找进程task\_struct

- 根据PID号（nr值）取得task\_struct 结构体

- 根据PID以及其类型（即为局部ID和命名空间）获取task\_struct结构体

如果根据的是**进程的ID号**，我们可以先通过ID号（nr值）获取到进程struct pid实体（局部ID），然后根据局部ID、以及命名空间，获得进程的task\_struct结构体

可以使用pid\_task()根据pid和pid\_type获取到进程的task\_struct

```c
struct task_struct *pid_task(struct pid *pid, enum pid_type type)
{
	struct task_struct *result = NULL;
	if (pid) {
		struct hlist_node *first;
		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
		lockdep_tasklist_lock_is_held());
		if (first)
			result = hlist_entry(first, struct task_struct, pids[(type)].node);
	}
    
	return result;
}
```

那么我们根据pid号查找进程task的过程就成为
```c
pTask = pid_task(find_vpid(pid), PIDTYPE_PID);  
```

内核还提供其它函数用来实现上面两步：

```c
struct task_struct *find_task_by_pid_ns(pid_t nr, struct pid_namespace *ns);
struct task_struct *find_task_by_vpid(pid_t vnr);
struct task_struct *find_task_by_pid(pid_t vnr);
```

>由于linux进程是组织在双向链表和红黑树中的，因此我们通过遍历链表或者树也可以找到当前进程，但是这个并不是我们今天的重点

## 4.4 生成唯一的PID

内核中使用下面两个函数来实现**分配和回收PID**的：

```c
static int alloc_pidmap(struct pid_namespace *pid_ns);
static void free_pidmap(struct upid *upid);
```

在这里我们不关注这两个函数的实现，反而应该关注分配的 PID 如何在多个命名空间中可见，这样需要在每个命名空间生成一个局部ID，函数 alloc\_pid 为新建的进程分配PID，简化版如下：

```c
struct pid *alloc_pid(struct pid_namespace *ns)
{
	struct pid *pid;
	enum pid_type type;
	int i, nr;
	struct pid_namespace *tmp;
	struct upid *upid;
	tmp = ns;
	pid->level = ns->level;
	// 初始化 pid->numbers[] 结构体
	for (i = ns->level; i >= 0; i--)
    {
		nr = alloc_pidmap(tmp); //分配一个局部ID
		pid->numbers[i].nr = nr;
		pid->numbers[i].ns = tmp;
		tmp = tmp->parent;
	}
	// 初始化 pid->task[] 结构体
	for (type = 0; type < PIDTYPE_MAX; ++type)
		INIT_HLIST_HEAD(&pid->tasks[type]);
	
    // 将每个命名空间经过哈希之后加入到散列表中
	upid = pid->numbers + ns->level;
	for ( ; upid >= pid->numbers; --upid)
    {
		hlist_add_head_rcu(&upid->pid_chain, &pid_hash[pid_hashfn(upid->nr, upid->ns)]);
    	upid->ns->nr_hashed++;
	}
    return pid;
}
```