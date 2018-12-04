[TOC]

# 1 进程描述符struct task\_struct

## 1.1 进程状态

```c
struct task_struct{
    volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
}
```

可能取值是

```c
[include/linux/sched.h]
 #define TASK_RUNNING            0
 #define TASK_INTERRUPTIBLE      1
 #define TASK_UNINTERRUPTIBLE    2
 #define __TASK_STOPPED          4
 #define __TASK_TRACED           8

/* in tsk->exit_state */
 #define EXIT_DEAD               16
 #define EXIT_ZOMBIE             32
 #define EXIT_TRACE              (EXIT_ZOMBIE | EXIT_DEAD)

/* in tsk->state again */
 #define TASK_DEAD               64
 #define TASK_WAKEKILL           128	/** wake on signals that are deadly **/
 #define TASK_WAKING             256
 #define TASK_PARKED             512
 #define TASK_NOLOAD             1024
 #define TASK_STATE_MAX          2048
 
 /* Convenience macros for the sake of set_task_state */
#define TASK_KILLABLE           (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED            (TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED             (TASK_WAKEKILL | __TASK_TRACED)
```

### 1.1.1 5个互斥状态

| 状态| 描述 | 
| ---- |:----|
| TASK\_RUNNING | 表示进程要么**正在执行**，要么正要**准备执行**（已经就绪），正在等待cpu时间片的调度 |
| TASK\_INTERRUPTIBLE | **阻塞态**.进程因为**等待一些条件**而**被挂起（阻塞**）而**所处的状态**。这些条件主要包括：**硬中断、资源、一些信号**……，**一旦等待的条件成立**，进程就会从该状态（阻塞）迅速**转化成为就绪状态TASK\_RUNNING** |
| TASK\_UNINTERRUPTIBLE | 意义与TASK\_INTERRUPTIBLE类似，除了**不能通过接受一个信号来唤醒**以外，对于处于TASK\_UNINTERRUPIBLE状态的进程，哪怕我们**传递一个信号或者有一个外部中断都不能唤醒他们**。**只有它所等待的资源可用**的时候，他才会被唤醒。这个标志很少用，但是并不代表没有任何用处，其实他的作用非常大，特别是对于驱动刺探相关的硬件过程很重要，这个刺探过程不能被一些其他的东西给中断，否则就会让进城进入不可预测的状态 |
| TASK\_STOPPED | 进程被**停止执行**，当进程接收到**SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号**之后就会进入该状态 |
| TASK\_TRACED | 表示**进程被debugger等进程监视**，进程执行被调试程序所停止，当一个进程被另外的进程所监视，每一个信号都会让进城进入该状态 |

### 1.1.2 2个终止状态

有**两个附加的进程状态**既可以**被添加到state域**中，又可以被添加到**exit\_state域**中.

只有**当进程终止**的时候，才会达到这两种状态.

```c
struct task_struct{
    int exit_state;
	int exit_code, exit_signal;
}
```

| 状态| 描述 | 
| ----- |:------|
| **EXIT\_ZOMBIE** | 进程的执行被终止，但是其**父进程还没有使用wait()等系统调用来获知它的终止信息**，此时进程成为**僵尸进程** |
| **EXIT\_DEAD** | 进程的**最终状态** |

### 1.1.3 睡眠状态

#### 1.1.3.1 内核将进程置为睡眠状态的方法

两种方法.

普通方法是将进程状态置为**TASK\_INTERRUPTIBLE**或**TASK\_UNINTERRUPTIBLE**, 然后**调用调度程序的schedule**()函数。这样会**将进程从CPU运行队列中移除**。

- **TASK\_INTERRUPTIBLE**: 可中断模式的睡眠状态, 可通过**显式的唤醒**呼叫(**wakeup\_process**())或者需要处理的**信号**来唤醒
- **TASK\_UNINTERRUPTIBLE**: 不可中断模式的睡眠状态, **只能**通过**显式的唤醒**呼叫, 一般不建议设置

新方法是使用新的进程睡眠状态TASK\_KILLABLE

TASK\_KILLABLE: 可以终止的新睡眠状态, 原理类似于TASK\_UNINTERRUPTIBLE，只不过**可以响应致命信号**

### 1.1.4 状态切换

进程状态的切换过程和原因大致如下图

![进程状态转换图](./images/1.gif)

## 1.2 进程标识符(PID)

```c
struct task_struct{
    pid_t pid;  
    pid_t tgid;
}
```

**pid来标识进程**，一个**线程组所有线程**与**领头线程**具有**相同的pid**，存入**tgid**字段，只有**线程组的领头线程**的**pid**成员才会被设置为**与tgid相同**的值。

注意, **getpid()返回当前进程的tgid值而不是pid的值(！！！**)。

在**CONFIG\_BASE\_SMALL配置为0**的情况下，**PID的取值范围是0到32767**，即系统中的**进程数最大为32768个**。

```cpp
#define PID_MAX_DEFAULT (CONFIG_BASE_SMALL ? 0x1000 : 0x8000)  
```

## 1.3 进程内核栈与thread\_info结构

### 1.3.1 为什么需要内核栈

```cpp
struct task_struct{
    // 指向内核栈的指针
    void *stack;
}
```

**进程**在**内核态运行**时需要自己的**堆栈信息**,因此linux内核为**每个进程(！！！每一个！！！**)都提供了一个**内核栈kernel stack(这里的stack就是这个进程在内核态的堆栈信息！！！**)

**内核态的进程**访问处于**内核数据段的栈**，这个栈**不同于**用户态的进程所用的栈。

**用户态进程**所用的**栈**，是在进程**线性地址空间**中；

而**内核栈**是当进程**从用户空间进入内核空间**时，**特权级发生变化**，需要**切换堆栈**，那么内核空间中使用的就是这个内核栈。因为内核控制路径使用**很少的栈空间**，所以**只需要几千个字节的内核态堆栈**。

需要注意的是，**内核态堆栈**仅用于**内核例程**，Linux内核另外为中断提供了单独的**硬中断栈**和**软中断栈**

### 1.3.2 为什么需要thread\_info

**内核**还需要存储**每个进程**的**PCB信息**,linux内核是**支持不同体系**的,但是**不同的体系结构**可能进程需要存储的**信息不尽相同**,这就需要我们实现一种**通用的方式**,我们将**体系结构相关**的部分和**无关**的部分进行**分离**

用一种**通用的方式**来描述进程, 这就是**struct task\_struct**, 而**thread\_info**就保存了**特定体系结构**的**汇编代码段**需要**访问的那部分进程的数据**, 我们在thread\_info中嵌入指向task\_struct的指针, 则我们可以很方便的**通过thread\_info**来**查找task\_struct**

### 1.3.3 内核栈和线程描述符

对**每个进程**，Linux内核都把**两个不同的数据结构紧凑**的存放在一个**单独为进程分配的内存区域**中

- 一个是**内核态**的**进程堆栈**，

- 另一个是紧挨着**进程描述符**的小数据结构thread\_info，叫做**线程描述符**。

Linux将这两个存放在一块, 这块区域通常是8192 Byte(两个页框), 其地址必须是8192的整数倍.

```c
[arch/x86/include/asm/page_32_types.h]
#define THREAD_SIZE_ORDER    1
// 2个页大小
#define THREAD_SIZE        (PAGE_SIZE << THREAD_SIZE_ORDER)
```

```c
[arch/x86/include/asm/page_64_types.h]
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER	(2 + KASAN_STACK_ORDER)
// 4个页或8个页大小
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

下图中显示了在**物理内存**中存放**两种数据结构**的方式。**线程描述符**驻留与这个**内存区的开始**，而**栈顶末端向下增长**。

![config](./images/2.png)

![config](./images/3.jpg)

**sp寄存器是CPU栈指针**，用来存放**栈顶单元的地址**。在80x86系统中，栈起始于顶端，并朝着这个内存区开始的方向增长。从用户态刚切换到内核态以后，进程的内核栈总是空的。因此，esp寄存器指向这个栈的顶端。一旦数据写入堆栈，esp的值就递减。

**进程描述符task\_struct**结构中**没有直接指向thread\_info结构的指针**，而是用一个**void指针类型**的成员表示，然后通过**类型转换来访问thread\_info结构**。

```cpp
#define task_thread_info(task)  ((struct thread_info *)(task)->stack)
```

### 1.3.4 内核栈数据结构描述thread\_info和thread\_union

**thread\_info是体系结构相关**的，结构的定义在thread\_info.h, 不同体系结构不同文件.

```c
[arch/x86/include/asm/thread_info.h]
struct thread_info {
	struct task_struct	*task;		/* main task structure */
	__u32			flags;		/* low level flags */
	__u32			status;		/* thread synchronous flags */
	__u32			cpu;		/* current CPU */
	mm_segment_t		addr_limit;
	unsigned int		sig_on_uaccess_error:1;
	unsigned int		uaccess_err:1;	/* uaccess failed */
};
```

Linux内核中使用一个**联合体**来表示一个**进程的线程描述符**和**内核栈**：

```c
[include/linux/sched.h]
union thread_union
{
	struct thread_info thread_info;
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

### 1.3.5 获取当前在CPU上正在运行进程的thread\_info

```cpp
static inline unsigned long current_top_of_stack(void)
{
#ifdef CONFIG_X86_64
    // 内核栈(0)栈顶寄存器SP0
	return this_cpu_read_stable(cpu_tss.x86_tss.sp0);
#else
	/* sp0 on x86_32 is special in and around vm86 mode. */
	return this_cpu_read_stable(cpu_current_top_of_stack);
#endif
}

static inline struct thread_info *current_thread_info(void)
{
	return (struct thread_info *)(current_top_of_stack() - THREAD_SIZE);
}
```

为了获取**当前CPU**上运行进程的**task\_struct**结构，内核提供了**current宏**，由于**task\_struct \*task**在**thread\_info的起始位置**，该宏本质上等价于**current\_thread\_info()\->task**

```c
[include/asm-generic/current.h]
#define get_current() (current_thread_info()->task)
#define current get_current()
```

### 1.3.6 分配和销毁thread\_info

进程通过**alloc\_thread\_info\_node**()函数**分配它的内核栈**，通过**free\_thread\_info**()函数释放所分配的内核栈。

## 1.4 进程标记

```c
struct task_struct{
    unsigned int flags;
}
```

反应**进程状态的信息**，但**不是运行状态**，用于**内核识别进程当前的状态**

取值以PF(ProcessFlag)开头的宏, 定义在include/linux/sched.h

## 1.5 表示进程亲属关系的成员

```c
struct task_struct{
    struct task_struct __rcu *real_parent; /* real parent process */
    struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
    
    struct list_head children;      /* list of my children */
    struct list_head sibling;       /* linkage in my parent's children list */
    struct task_struct *group_leader;       /* threadgroup leader */
}
```

| 字段 | 描述 |
| --- |:---|
| real\_parent | 指向其父进程，如果创建它的**父进程不再存在**，则指向**PID为1的init进程** |
| parent | 指向其父进程，**当它终止时，必须向它的父进程发送信号**。它的值通常与real\_parent相同 |
| children | 表示**链表的头部**，链表中的**所有元素**都是它的**子进程** |
| sibling | 用于**把当前进程插入到兄弟链表**中 |
| group\_leader | 指向其所在**进程组的领头进程** |

## 1.6 ptrace系统调用

ptrace提供了一种**父进程可以控制子进程运行**，并可以检查和改变它的核心image。

主要用于**实现断点调试**。一个被跟踪的进程运行中，直到发生一个**信号**,则**进程被中止**，并且**通知其父进程**。在**进程中止的状态**下，进程的**内存空间可以被读写**。父进程还可以使子进程继续执行，并选择是否是否忽略引起中止的信号。

```c
struct task_struct{
    unsigned int ptrace;
    struct list_head ptraced;
    struct list_head ptrace_entry;
    
    unsigned long ptrace_message;
    siginfo_t *last_siginfo;
}
```

成员**ptrace被设置为0**时表示**不需要被跟踪**, 取值定义在文件include/linux/ptrace.h, 以PT开头

## 1.7 Performance Event

**性能诊断工具**.分析进程的性能问题.

```c
struct task_struct{
#ifdef CONFIG_PERF_EVENTS
    struct perf_event_context *perf_event_ctxp[perf_nr_task_contexts];
    struct mutex perf_event_mutex;
    struct list_head perf_event_list;
#endif
}
```

## 1.8 进程调度

### 1.8.1 优先级

```c
struct task_struct{
    int prio, static_prio, normal_prio;
    unsigned int rt_priority;
```

| 字段 | 描述 |
| ------------- |:-------------:|
| static\_prio | 用于保存**静态优先级**，可以通过**nice系统调用**来进行修改 |
| rt\_priority | 用于保存**实时优先级** |
| normal\_prio | 值取决于**静态优先级和调度策略** |
| prio | 用于保存**动态优先级** |

**实时优先级**范围是0到MAX\_RT\_PRIO\-1（即99），而**普通进程**的**静态优先级范围**是从**MAX\_RT\_PRIO到MAX\_PRIO-1（即100到139**）。**值越大静态优先级越低**。

### 1.8.2 调度策略相关字段

```c
struct task_struct{
    unsigned int policy;
    const struct sched_class *sched_class;
    struct sched_entity se;
    struct sched_rt_entity rt;
    cpumask_t cpus_allowed;
}
```

| 字段 | 描述 |
| ------------- |:-------------:|
| policy | **调度策略** |
| sched\_class | **调度类** |
| se | **普通进程**的调度实体，每个进程都有其中之一的实体 |
| rt | **实时进程**的调度实体，每个进程都有其中之一的实体 |
| cpus\_allowed | 用于控制进程可以在**哪些处理器**上运行 |

## 1.9 进程地址空间

```c
struct task_struct{
    struct mm_struct *mm, *active_mm;
    /* per-thread vma caching */
    u32 vmacache_seqnum;
    struct vm_area_struct *vmacache[VMACACHE_SIZE];
    #if defined(SPLIT_RSS_COUNTING)
    struct task_rss_stat    rss_stat;
    #endif
    
    #ifdef CONFIG_COMPAT_BRK
    unsigned brk_randomized:1;
    #endif
}
```

| 字段 | 描述 |
| ---- |:---|
| mm | 进程**所拥有的用户空间内存描述符(拥有的！！！**)，**内核线程**无,mm为**NULL** |
| active\_mm | active\_mm指向**进程运行时所使用的内存描述符(使用的！！！内核线程不拥有用户空间内存,但是必须有使用的空间**)，对于**普通进程**而言，这两个指针变量的值相同。但是**内核线程kernel thread是没有进程地址空间**的，所以**内核线程的tsk->mm域是空（NULL**）。但是**内核必须知道用户空间包含了什么**，因此它的active\_mm成员被初始化为**前一个运行进程的mm**值。|
| brk\_randomized| 用来确定**对随机堆内存的探测**。参见[LKML]( http://lkml.indiana.edu/hypermail/linux/kernel/1104.1/00196.html)上的介绍 |
| rss\_stat | 用来**记录缓冲信息** |

如果**当前内核线程**被**调度之前**运行的也是**另外一个内核线程**时候，那么其**mm和avtive\_mm都是NULL**

对Linux来说，用户进程和内核线程（kernel thread)都是task\_struct的实例，唯一的区别是**kernel thread**是**没有进程地址空间**的，**内核线程**也**没有mm描述符**的，所以内核线程的tsk\->mm域是空（NULL）。

内核scheduler在进程context switching的时候，会**根据tsk\->mm**判断即将调度的进程是**用户进程**还是**内核线程**。

但是虽然**thread thread**不用访问**用户进程地址空间**，但是**仍然需要page table**来访问**kernel自己的空间**。但是幸运的是，对于**任何用户进程**来说，他们的**内核空间都是100%相同**的，所以内核可以’borrow'上一个被调用的**用户进程的mm中的页表**来访问**内核地址**，这个mm就记录在active\_mm。

简而言之就是，对于kernel thread,tsk\->mm == NULL表示自己内核线程的身份，而tsk\->active\_mm是借用上一个用户进程的mm，用mm的page table来访问内核空间。对于**用户进程**，tsk\->mm == tsk\->active\_mm。

## 1.10 判断标志

```c
struct task_struct{
    int exit_code, exit_signal;
    int pdeath_signal;  /*  The signal sent when the parent dies  */
    unsigned long jobctl;   /* JOBCTL_*, siglock protected */
     
    /* Used for emulating ABI behavior of previous Linux versions */
    unsigned int personality;
     
    /* scheduler bits, serialized by scheduler locks */
    unsigned sched_reset_on_fork:1;
    unsigned sched_contributes_to_load:1;
    unsigned sched_migrated:1;
    unsigned :0; /* force alignment to the next boundary */
     
    /* unserialized, strictly 'current' */
    unsigned in_execve:1; /* bit to tell LSMs we're in execve */
    unsigned in_iowait:1;
}
```

| 字段 | 描述 |
| ------------- |:-------------|
| exit\_code | 用于设置**进程的终止代号**，这个值要么**是\_exit()或exit\_group()系统调用参数**（**正常终止**），要么是由**内核提供的一个错误代号（异常终止**）。|
| exit\_signal | 被置**为-1**时表示是**某个线程组中的一员**。只有当**线程组**的**最后一个成员终止**时，才会**产生一个信号**，以**通知线程组的领头进程的父进程**。|
| pdeath\_signal | 用于**判断父进程终止时发送信号**。|
| personality | 用于处理不同的ABI |
| in\_execve | 用于通知LSM是否被do\_execve()函数所调用 |
| in\_iowait | 用于判断是否**进行iowait计数** |
| sched\_reset\_on\_fork | 用于判断是否**恢复默认的优先级或调度策略** |

## 1.11 时间

| 字段 | 描述 |
| ---- |:---|
| utime/stime | 用于记录进程在**用户态/内核态**下所经过的**节拍数（定时器**）| 
| prev\_utime/prev\_stime | **先前的运行时间** |
| utimescaled/stimescaled | 用于记录进程在**用户态/内核态的运行时间**，但它们**以处理器的频率**为刻度 |
| gtime | 以**节拍计数**的**虚拟机运行时间**（guest time） |
| nvcsw/nivcsw | 是**自愿（voluntary）/非自愿（involuntary）上下文切换计数** |
| last\_switch\_count | nvcsw和nivcsw的总和 |
| start\_time/real\_start\_time | 进程**创建时间**，real\_start\_time还包含了**进程睡眠时间**，常用于/proc/pid/stat |
| cputime\_expires | 用来统计**进程或进程组被跟踪的处理器时间**，其中的三个成员对应着cpu\_timers\[3\]的三个链表 |

## 1.12 信号处理

```c
struct task_struct{
    struct signal_struct *signal;
    struct sighand_struct *sighand;
    sigset_t blocked, real_blocked;
    sigset_t saved_sigmask; /* restored if set_restore_sigmask() was used */
    struct sigpending pending;
    unsigned long sas_ss_sp;
    size_t sas_ss_size;
}
```

| 字段 | 描述 |
| --- |:---|
| signal | 指向进程的**信号描述符** |
| sighand | 指向进程的**信号处理程序描述符** |
| blocked | 表示**被阻塞信号的掩码**，real\_blocked表示临时掩码 |
| pending | 存放**私有挂起信号**的数据结构 |
| sas\_ss\_sp | 是**信号处理程序备用堆栈的地址**，sas\_ss\_size表示**堆栈的大小** |

## 1.13 其他

# 2 命名空间

Linux Namespaces机制提供一种**资源隔离**方案。

**命名空间**是为**操作系统层面的虚拟化机制**提供支撑，目前实现的有**六种不同的命名空间**，分别为**mount命名空间、UTS命名空间、IPC命名空间、用户命名空间、PID命名空间、网络命名空间**。命名空间简单来说提供的是对**全局资源的一种抽象**，将资源放到不同的容器中（不同的命名空间），各容器彼此隔离。

命名空间**有的**还有**层次关系**，如**PID命名空间**

要**创建新的Namespace**，只需要在**调用clone时指定相应的flag**。

**LXC（Linux containers**）就是利用这一特性实现了资源的隔离。

虽然**子容器不了解系统中的其他容器**，但**父容器知道子命名空间的存在**，也**可以看到其中执行的所有进程**。**子容器的进程映射到父容器**中，PID为4到9。尽管系统上有9个进程，但却需要15个PID来表示，因为**一个进程可以关联到多个PID**。

## 2.1 Linux内核命名空间描述

在Linux内核中提供了**多个namespace！！！**，**一个进程可以属于多个namesapce**.

在task\_struct 结构中有一个指向**namespace结构体的指针nsproxy**。

```c
struct task_struct
{
    /* namespaces */
    struct nsproxy *nsproxy;
}

struct nsproxy
{
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net           *net_ns;
};
```

struct nsproxy定义了5个指向各个类型namespace的指针, 由于**多个进程可以使用同一个namespace**，所以nsproxy可以共享使用，**count字段是该结构的引用计数**。

1. UTS命名空间包含了**运行内核的名称、版本、底层体系结构类型等信息**。UTS是**UNIX Timesharing System**的简称。

2. 保存在struct ipc\_namespace中的所有与**进程间通信（IPC）有关的信息**。

3. 已经装载的**文件系统的视图**，在struct mnt\_namespace中给出。

4. 有关**进程ID的信息**，由struct pid\_namespace提供。

5. struct net包含所有**网络相关**的命名空间参数。

系统中有一个**默认的nsproxy**，**init\_nsproxy**，该结构**在task初始化是也会被初始化**，定义在include/linux/init\_task.h

```c
#define INIT_TASK(tsk)  \
{
    .nsproxy = &init_nsproxy,
}
```

其中init\_nsproxy的定义为：
 
```
struct nsproxy init_nsproxy = {
         .count                         = ATOMIC_INIT(1),
         .uts_ns                       = &init_uts_ns,
#if defined(CONFIG_POSIX_MQUEUE) || defined(CONFIG_SYSVIPC)
         .ipc_ns                        = &init_ipc_ns,
#endif
         .mnt_ns                      = NULL,
         .pid_ns_for_children        = &init_pid_ns,
#ifdef CONFIG_NET
         .net_ns                       = &init_net,
#endif
};
```
对于.**mnt\_ns没有进行初始化**，其余的namespace都进行了系统默认初始化

## 2.2 命名空间的创建

新的命名空间可以用下面两种方法创建。

1. 在用**fork或clone系统调用创建新进程**时，有**特定的选项**可以控制是**与父进程共享命名空间**，还是建立**新的命名空间**。

2. **unshare系统调用**将进程的某些部分**从父进程分离**，其中也包括**命名空间**。

**命名空间的实现**需要两个部分：

- **每个子系统的命名空间结构**，将此前所有的**全局组件包装到命名空间**中；

- 将**给定进程关联到所属各个命名空间的机制**。

使用fork或clone系统调用创建新进程可使用的选项:

- CLONE\_NEWPID    **进程命名空间**。空间内的PID是独立分配的，意思就是命名空间内的虚拟PID可能会与命名空间外的PID相冲突，于是**命名空间内的PID映射到命名空间外时会使用另外一个PID**。比如说，命名空间内第一个PID为1，而在命名空间外就是该PID已被init进程所使用。
    
- CLONE\_NEWIPC    **进程间通信(IPC)的命名空间**，可以将SystemV的IPC和POSIX的消息队列独立出来。

- CLONE\_NEWNET    网络命名空间，用于隔离网络资源（/proc/net、IP地址、网卡、路由等）。后台进程可以运行在不同命名空间内的相同端口上，用户还可以虚拟出一块网卡。

- CLONE\_NEWNS     **挂载命名空间**，进程运行时可以将挂载点与系统分离，使用这个功能时，我们可以达到 chroot 的功能，而在安全性方面比 chroot 更高。
    
- CLONE\_NEWUTS    **UTS命名空间**，主要目的是独立出主机名和网络信息服务（NIS）。

- CLONE\_NEWUSER   **用户命名空间**，同进程ID一样，**用户ID**和**组ID**在命名空间内外是不一样的，并且在不同命名空间内可以存在相同的ID。

## 2.3 PID Namespace

**CLONE\_NEWPID**, 会创建一个新的PID Namespace，clone出来的**新进程**将成为**Namespace里的第一个进程**，**PID Namespace内的PID将从1开始**, 类似于独立系统中的init进程, 该Namespace内的**孤儿进程都将以该进程为父进程**，当该进程被结束时，该Namespace内所有的进程都会被结束。

**PID Namespace是层次性**，新创建的Namespace将会是创建该Namespace的进程属于的Namespace的**子Namespace**。子Namespace中的**进程**对于**父Namespace是可见的**，**一个进程**将拥有**不止一个PID**，而是在**所在的Namespace**以及**所有直系祖先Namespace**中都**将有一个PID(直系祖先！！！**)。

系统启动时，内核将创建一个**默认的PID Namespace**，该Namespace是所有以后创建的Namespace的祖先，因此**系统所有的进程在该Namespace都是可见**的。

# 3 进程ID类型

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
- **PID**, 其**命名空间**中**唯一标识进程**
- **TGID**, **线程组（轻量级进程组**）的ID标识

在一个进程中，如果以**CLONE\_THREAD标志**来调用clone建立的进程就是**该进程的一个线程**（即**轻量级进程**，Linux其实**没有严格的线程概念**），它们处于一个线程组, 所有进程都有相同的TGID, pid不同.

**线程组组长（也叫主线程**）的TGID与其PID相同；一个**进程没有使用线程**，则其TGID与PID也**相同**。

该枚举没有包括线程组ID, 因为task\_struct已经有指向线程组的指针

```c
struct task_struct{
    struct task_struct *group_leader;
}
```

- **PGID**, **进程组**的ID标识

**独立的进程可以组成进程组**（使用**setpgrp系统调用**），进程组可以简化向所有组内进程发送信号的操作

- **SID**, **会话组**的ID标识

**几个进程组**可以合并成**一个会话组**（使用**setsid系统调用**）, SID保存在**task\_struct的session**成员中

# 4 PID命名空间

## 4.1 pid命名空间概述

**PID命名空间**有**层次关系**

![config](./images/4.jpg)

上图有四个命名空间，一个父命名空间衍生了两个子命名空间，其中的一个子命名空间又衍生了一个子命名空间。以PID命名空间为例，由于各个命名空间彼此隔离，所以每个命名空间都可以有 PID 号为 1 的进程；但又由于命名空间的层次性，父命名空间是知道子命名空间的存在，因此子命名空间要映射到父命名空间中去，因此上图中 level 1 中两个子命名空间的六个进程分别映射到其父命名空间的PID号5\~10。

level 2的PID是1的进程在level 1和level 0都有映射

## 4.2 局部ID和全局ID

**全局ID**: 在内核本身和**初始命名空间中唯一的ID(初始命名空间中！！！**), 系统启动期间开始的init进程即属于该初始命名空间。

**局部ID**: 属于**某个特定的命名空间**

### 4.2.1 全局ID

- **全局PID和全局TGID**直接保存在task\_struct中，分别是task\_struct的**pid和tgid成员**：

```c
<sched.h> 
struct task_struct
{
    pid_t pid;  
    pid_t tgid; 
}
```

两项都是**pid\_t类型**，该**类型定义为\_\_kernel\_pid\_t**，后者由**各个体系结构分别定义**。通常定义为int，即可以同时使用232个不同的ID。

- **task\_struct->signal->\_\_session**表示**全局SID**, **set\_task\_session**用于修改

- **全局PGID**则保存在**task\_struct->signal->\_\_pgrp**, **set\_task\_pgrp**用于修改

### 4.2.2 局部ID

## 4.3 PID命名空间数据结构pid\_namespace

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

| 字段| 描述 | 
| ------------- |:-------------|
| kref | 表示**指向pid\_namespace的个数** |
| pidmap | pidmap结构体表示**分配pid的位图**。当需要分配一个新的pid时只需查找位图，找到**bit为0的位置并置1**，然后**更新统计数据域**（nr\_free) |
| last\_pid | 用于pidmap的分配。指向最后一个分配的pid的位置。 |
| child\_reaper | 指向的是**当前命名空间的init进程**，每个命名空间都有一个作用相当于全局init进程的进程 |
| pid\_cachep | 域指向**分配pid的slab的地址**。|
| level | 代表**当前命名空间的等级**，**初始命名空间的level为0**，它的子命名空间level为1，依次递增，而且子命名空间对父命名空间是可见的。从**给定的level**设置，内核即可推断**进程会关联到多少个ID**。|
| parent | 指向**父命名空间的指针** |

![config](./images/5.png)

PID分配器也需要依靠该结构的某些部分来连续生成唯一ID

**每个PID命名空间都具有一个进程**，其发挥的作用**相当于全局的init进程**。init的一个目的是**对孤儿进程调用wait4**，命名空间**局部的init变体**也必须完成该工作。

# 5 pid结构描述

## 5.1 pid与upid

**PID的管理**围绕两个数据结构展开：

- struct pid是**内核对PID的内部表示**

- struct upid则表示**特定的命名空间中可见**的信息

### 5.1.1 特定命名空间信息struct upid

```c
[include/linux/pid.h]
struct upid
{
    int nr;  
    struct pid_namespace *ns;  
    struct hlist_node pid_chain;  
};  
```

struct upid是一个**特定namespace**里面的进程的信息,包含该namespace里面进程具体ID号,namespace指针,哈希列表指针.

| 字段| 描述 | 
| ---- |:------|
| nr | 表示**在该命名空间所分配的进程ID具体的值** |
| ns | 指向**命名空间的指针** |
| pid\_chain | 指向**PID哈希列表的指针**，用于**关联对应的PID** |

**所有的upid实例**都保存在一个**散列表**中

### 5.1.2 局部ID类struct pid

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

tasks是一个数组，**每个数组项**都是一个**散列表头**，对应于一个ID类型, PIDTYPE\_PID,PIDTYPE\_PGID,PIDTYPE\_SID（PIDTYPE\_MAX表示**ID类型的数目**）这样做是必要的，因为**一个ID可能用于几个进程(task\_struct)！！！**。所有**共享同一ID**的**task\_struct实例**，都**通过该列表连接起来(这个列表就是使用这个pid的进程<task\_struct>的列表！！！**)。

## 5.2 用于分配pid的位图struct pidmap

需要分配一个新的pid时查找可使用pid的位图

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

## 5.3 pid的哈希表存储结构struct pid\_link

task\_struct中的struct pid\_link pids[PIDTYPE\_MAX]指向了**和该task\_struct相关的pid结构体**。

```c
struct pid_link  
{  
    struct hlist_node node;  
    struct pid *pid;  
};
```

## 5.4 task\_struct中的进程ID相关描述符信息

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
    pid_t pid;
    pid_t tgid;
    struct task_struct *group_leader;
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

task\_struct结构信息

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

![进程命名空间和进程的关系结构](./images/6.png)

可以看到，**多个task\_struct**指向**一个PID**，同时PID的hash数组里安装不同的类型对task进行散列，并且**一个PID**会属于多个命名空间。

- 进程的结构体是task\_struct,**一个进程对应一个task\_struct结构体(一对一**).**一个进程**会有**PIDTYPE\_MAX个(3个)pid\_link结构体(一对多**),这**三个结构体中的pid**分别指向 ①该进程对应的**进程本身(PIDTYPE\_PID**)的真实的pid结构体; ②该进程的**进程组(PIDTYPE\_PGID)的组长本身**的pid结构体; ③该进程的**会话组(PIDTYPE\_SID)的组长**本身的pid结构体. 所以**一个真实的进程只会有一个自身真实的pid结构体**
- 这三个pid\_link结构体里面有个**哈希节点node**,因为进程组、会话组等的存在,这个**node用来链接同一个组的进程task\_struct**,指向的是**task\_struct**中的pid\_link的node
- pid结构体(不是一个ID号)代表**一个真实的进程(某个组的组长的pid也是这个结构体,因为组长也是真实的进程,也就有相应的真实的pid结构体,而组长身份是通过task\_struct引的**),所以里面会有 ①**该进程真实所处命名空间的level**; ②**PIDTYPE\_MAX个(3个)散列表头**,tasks[PIDTYPE\_PID]指向自身进程(因为PIDTYPE\_PID是PID类型),如果该进程是进程组组长,那么tasks[PIDTYPE\_PGID]就是这个散列表的表头,指向下一个进程的相应组变量pids[PIDTYPE\_PGID]的node,如果该进程是会话组组长,那么tasks[PIDTYPE\_SID]就是这个散列表的表头,指向下一个进程的相应组变量pids[PIDTYPE\_SID]的node; ③由于一个进程可能会呈现在多个pid命名空间,所以有该进程在其他命名空间中的信息结构体upid的数组,每个数组项代表一个
- 结构体upid的数组number[1],**数组项个数取决于该进程pid的level值**,**每个数组项代表一个命名空间**,这个就是用来一个PID可以属于不同的命名空间,nr值表示该进程在该命名空间的pid值,ns指向该信息所在的命名空间,pid\_chain属于哈希表的节点. 系统有一个**pid\_hash**[], 通过**pid**在**某个命名空间的nr值**哈希到某个表项,如果**多个nr值**哈希到**同一个表项**,将其**加入链表**,这个节点就是**upid的pid\_chain**

![增加PID命名空间之后的结构图](./images/task_struct-with-namespace.png)

图中关于**如何分配唯一的PID没有画出**

## 5.5 进程ID管理函数

### 5.5.1 进程pid号找到struct pid实体

首先就需要通过**进程的pid找到进程的struct pid**，然后再**通过struct pid找到进程的task\_struct**

实现函数有三个

```c
//通过pid值找到进程的struct pid实体
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
struct pid *find_vpid(int nr)
struct pid *find_get_pid(pid_t nr)
```

find\_pid\_ns获得pid实体的实现原理，**主要使用哈希查找**。

内核使用**哈希表组织struct pid**，每创建一个**新进程**，给进程的struct pid都会**插入到哈希表**中，这时候就需要使用进程的**进程pid**和pid命名空间ns在哈希表中将相对应的struct pid索引出来

根据**局部PID**以及**命名空间**计算在**pid\_hash数组中的索引**，然后**遍历散列表**找到**所要的upid**，再根据内核的 container\_of 机制找到 pid 实例。

```c
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
        struct hlist_node *elem;
        struct upid *pnr; 
        //遍历散列表
        hlist_for_each_entry_rcu(pnr, elem,
            &pid_hash[pid_hashfn(nr, ns)], pid_chain) //pid_hashfn() 获得hash的索引 
                // 比较 nr 与 ns 是否都相同
                if (pnr->nr == nr && pnr->ns == ns) 
                    //根据container_of机制取得pid 实体
                    return container_of(pnr, struct pid, numbers[ns->level]);
        return NULL;
}
EXPORT_SYMBOL_GPL(find_pid_ns);
```

### 5.5.2 获取局部ID

根据进程的 task\_struct、ID类型、命名空间，可以很容易获得其在命名空间内的局部ID

### 5.5.3 根据PID查找进程task\_struct

- 根据PID号（nr值）取得task\_struct 结构体

- 根据PID以及其类型（即为局部ID和命名空间）获取task\_struct结构体

如果根据的是**进程的ID号**，我们可以先通过ID号（nr值）获取到进程struct pid实体（局部ID），然后根据局部ID、以及命名空间，获得进程的task\_struct结构体

### 5.5.4 生成唯一的PID

内核中使用下面两个函数来实现**分配和回收PID**的：

```c
static int alloc_pidmap(struct pid_namespace *pid_ns);
static void free_pidmap(struct upid *upid);
```

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

# 6 Liux进程类别

Linux下**只有一种类型**的进程，那就是**task\_struct**，当然我也想说**linux其实也没有线程的概念**, 只是将那些**与其他进程共享资源的进程称之为线程**。

通常在**一个进程**中可以包含**若干个线程**，它们可以**利用进程所拥有的资源**。通常把**进程作为分配资源的基本单位**，而把**线程作为独立运行和独立调度的基本单位**。

线程和进程的区别在于，**子进程和父进程有不同的代码和数据空间**，而**多个线程则共享数据空间**，**每个线程有自己的执行堆栈和程序计数器为其执行上下文(！！！这些是线程独享的！！！**)。

1.	一个进程**由于其运行空间的不同**, 从而有**内核线程**和**用户进程**的区分, **内核线程运行在内核空间**, 之所以称之为**线程**是**因为它没有虚拟地址空间(唯一使用的资源是内核栈和上下文切换时保持寄存器的空间**), 只能访问**内核的代码和数据**, 而用户进程则运行在**用户空间**, 但是可以通过**中断,系统调用等方式从用户态陷入内核态**。

2.	**用户进程**运行在用户空间上,而一些通过**共享资源实现的一组进程**我们称之为**线程组**, Linux下内核其实本质上没有线程的概念,**Linux**下**线程**其实上是**与其他进程共享某些资源的进程**而已。但是我们习惯上还是称他们为**线程**或者**轻量级进程**

因此, **Linux上进程**分3种，**内核线程**（或者叫**内核进程**）、**用户进程**、**用户线程(！！！因为内核里面的进程没有虚拟地址空间！！！**), 当然如果更严谨的，你也可以认为**用户进程和用户线程都是用户进程**。

- **内核线程拥有进程描述符、PID、进程正文段、内核堆栈**

- **用户进程拥有进程描述符、PID、进程正文段、内核堆栈、用户空间的数据段和堆栈**

- **用户线程拥有进程描述符、PID、进程正文段、内核堆栈，同父进程共享用户空间的数据段和堆栈**

**用户线程**也可以通过**exec函数族**拥有自己的**用户空间的数据段和堆栈**，成为**用户进程**。

进程task\_struct中**pid存储的是内核对该进程的唯一标示**, 即对**进程**则标示**进程号**, 对**线程**来说就是其**线程号**, 那么对于**线程**来说**一个线程组所有线程与领头线程具有相同的进程号，存入tgid字段**

每个线程除了共享进程的资源外还拥有各自的私有资源：一个寄存器组（或者说是线程上下文）；一个专属的堆栈；一个专属的消息队列；一个专属的Thread Local Storage（TLS）；一个专属的结构化异常处理串链。

## 6.1 内核线程

**只运行在内核态**，不受**用户态上下文**的拖累。

从**内核的角度**来说, Linux并**没有线程这个概念**。Linux把**所有的线程都当做进程**来实现。

跟普通进程一样，**内核线程也有优先级和被调度**。当和**用户进程**拥有**相同的static\_prio**时，内核线程有机会得到更多的cpu资源

**内核线程没有自己的地址空间**，所以它们的"**current\->mm**"都是**空的**, **唯一使用的资源**就是**内核栈**和**上下文切换时保存寄存器的空间**。

**内核线程还有核心堆栈**，没有mm怎么访问它的核心堆栈呢？这个**核心堆栈跟task\_struct的thread\_info共享8k的空间**，所以不用mm描述。

但是**内核线程**总要**访问内核空间的其他内核**啊，没有mm域毕竟是不行的。所以内核线程被调用时,内核会将其task\_strcut的**active\_mm**指向**前一个被调度出的进程的mm域**,在需要的时候，内核线程可以使用前一个进程的内存描述符。

因为**内核线程不访问用户空间**，**只操作内核空间内存**，而**所有进程的内核空间都是一样的**。这样就省下了一个mm域的内存。

# 7 linux进程的创建流程

## 7.1 进程的复制fork和加载execve

Linux下进行进行编程，往往都是通过fork出来一个新的程序.

**一个进程**，包括**代码、数据和分配给进程的资源**，它其实是从**现有的进程（父进程**）复制出的一个副本（子进程），**fork**()函数通过**系统调用**创建一个与原来进程几乎完全相同的进程，也就是两个进程可以做完全相同的事，然后如果我们通过**execve为子进程加载新的应用程序**后，那么新的进程将开始执行新的应用

- fork生成当前进程的的一个相同副本，该副本成为子进程

原进程（父进程）的所有资源都以适当的方法复制给新的进程（子进程）。因此该系统调用之后，原来的进程就有了**两个独立的实例**，这两个实例的联系包括：**同一组打开文件**,**同样的工作目录**,**进程虚拟空间（内存）中同样的数据**（当然两个进程各有一份副本,也就是说他们的**虚拟地址相同**,但是所对应的**物理地址不同**）等等。

- execve从一个可执行的二进制程序镜像加载应用程序, 来代替当前运行的进程

换句话说, 加载了一个**新的应用程序**。因此execv并不是创建新进程

所以我们在linux要创建一个进程的时候，其实执行的操作就是

1. 首先使用fork复制一个旧的进程

2. 然后调用execve在为新的进程加载一个新的应用程序

## 7.2 写时复制技术

大批量的复制会导致执行效率过低。

现在的Linux内核采用一种更为有效的方法，称之为写时复制（Copy On Write，COW）。这种思想相当简单：**父进程和子进程共享页帧而不是复制页帧**。然而，**只要页帧被共享**，它们就不能被修改，即**页帧被保护**。无论父进程还是子进程何时试图**写一个共享的页帧**，就**产生一个异常**，这时**内核就把这个页复制到一个新的页帧中并标记为可写(标志位设置只是对用户特权级即3特权级有效**)。原来的页帧仍然是写保护的：当其他进程试图写入时，内核检查写进程是否是这个页帧的唯一属主，如果是，就把**这个页帧**标记为**对这个进程**是**可写的**。

当父进程A或子进程B任何一方对这些已共享的物理页面执行写操作时,都会产生**页面出错异常(page\_fault int14)中断**,此时CPU会执行系统提供的**异常处理函数do\_wp\_page**()来解决这个异常.

**do\_wp\_page**()会对这块导致写入异常中断的**物理页面**进行**取消共享**操作,**为写进程复制一新的物理页面**,使父进程A和子进程B各自拥有一块**内容相同的物理页面**.最后,从**异常处理函数中返回**时,CPU就会**重新执行刚才导致异常的写入操作指令**,使进程继续执行下去.

一个进程调用fork()函数后，系统先给**新的进程分配资源**，例如存储数据和代码的空间。然后把原来的进程的所有值都复制到新的新进程中，只有**少数值**与原来的进程的值（比如**PID**）不同。相当于克隆了一个自己。

## 7.3 内核线程创建接口

在**内核**中，有两种方法可以**生成内核线程**，一种是使用**kernel\_thread**()接口，另一种是用**kthread\_create**()接口

### 7.3.1 kernel\_thread

先说kernel\_thread接口，使用该接口创建的线程，必须在该线程中**调用daemonize**()函数，这是因为只有**当线程的父进程指向"Kthreadd**"时，该线程**才算是内核线程(！！！**)，而恰好**daemonize**()函数主要工作便是**将该线程的父进程改成“kthreadd**"内核线程；

默认情况下，**调用deamonize**()后，会**阻塞所有信号**，如果想操作某个信号可以调用**allow\_signal**()函数。

```c
// fn为线程函数，arg为线程函数参数，flags为标记
int kernel_thread(int (*fn)(void *), void *arg, unsigned long flags); 
// name为内核线程的名称
void daemonize(const char * name,...); 
```

### 7.3.2 kthread\_create

而kthread\_create接口，则是**标准**的内核线程创建接口，只须调用该接口便可创建内核线程；

**默认创建的线程**是存于**不可运行的状态**，所以需要**在父进程中**通过**调用wake\_up\_process**()函数来启动该线程。

```c
//threadfn为线程函数;data为线程函数参数;namefmt为线程名称，可被格式化的, 类似printk一样传入某种格式的线程名
struct task_struct *kthread_create(int (*threadfn)(void *data),void *data,
                                  const char namefmt[], ...);
 
```

线程创建后，不会马上运行，而是需要将**kthread\_create**()返回的**task\_struct指针**传**给wake\_up\_process**()，然后通过此函数运行线程。

### 7.3.3 kthread\_run

当然，还有一个创建并启动线程的函数：kthread\_run

```c
struct task_struct *kthread_run(int (*threadfn)(void *data),
                                    void *data,
                                    const char *namefmt, ...);
```

线程一旦启动起来后，会**一直运行**，除非该线程主动调用**do\_exit函数**，或者**其他的进程**调用**kthread\_stop函数**，结束线程的运行。

```c
int kthread_stop(struct task_struct *thread);
```

kthread\_stop() 通过**发送信号**给线程。

如果线程函数正在处理一个非常重要的任务，它不会被中断的。当然如果线程函数永远不返回并且不检查信号，它将永远都不会停止。

```c
//唤醒线程
int wake_up_process(struct task_struct *p); 
//是以上两个函数的功能的总和
struct task_struct *kthread_run(int (*threadfn)(void *data),void *data,
                                const char namefmt[], ...);
```

因为**线程也是进程**，所以其结构体也是使用进程的结构体"struct task\_struct"。

## 7.4 内核线程的退出接口

当**内核线程执行到函数末尾**时会**自动调用内核中do\_exit**()函数来退出或其他线程调用**kthread\_stop**()来指定线程退出。

怎么调用do\_exit()可以看kthreadd线程

```c
    int kthread_stop(struct task_struct *thread);
```

kthread\_stop()通过**发送信号给线程**。

如果**线程函数**正在处理一个非常重要的任务，它**不会被中断**的。当然如果**线程函数永远不返回并且不检查信号**，它将**永远都不会停止**。

在**执行kthread\_stop的时候**，目标线程**必须没有退出**，否则**会Oops**。原因很容易理解，当**目标线程退出**的时候，其对应的**task结构也变得无效**，kthread\_stop**引用该无效task结构就会出错**。

为了避免这种情况，**需要确保线程没有退出**，其方法如代码中所示：

```c
thread_func()
{
    // do your work here
    // wait to exit
    while(!thread_could_stop())
    {
           wait();
    }
}

exit_code()
{
     kthread_stop(_task);   //发信号给task，通知其可以退出了
}
```
这种退出机制很温和，一切尽在thread\_func()的掌控之中，**线程在退出**时可以从容地**释放资源**，而不是莫名其妙地被人“暗杀”。

# 8 Linux中3个特殊的进程

Linux下有**3个特殊的进程**，**idle**进程($**PID = 0**$), **init**进程($**PID = 1**$)和**kthreadd**($**PID = 2**$)

- idle进程由**系统自动创建**, 运行在**内核态**

idle进程其pid=0，其前身是**系统创建的第一个进程**，也是**唯一一个没有通过fork或者kernel\_thread产生的进程**。完成加载系统后，演变为**进程调度、交换**

- **init**进程由**idle**通过**kernel\_thread创建**，在**内核空间（！！！）完成初始化后**, 最终**执行/sbin/init进程**，变为**所有用户态程序的根进程（pstree命令显示**）,即**用户空间的init进程**

由**0进程创建**，完成**系统的初始化**.是系统中**所有其它用户进程（！！！用户进程！！！）的祖先进程**.Linux中的**所有进程**都是有**init进程创建并运行**的。首先Linux内核启动，然后**在用户空间中启动init进程**，再**启动其他系统程**。在**系统启动完成后**，**init**将变为**守护进程监视系统其他进程**

- kthreadd进程由**idle**通过**kernel\_thread**创建,并始终运行在**内核空间**,负责所有**内核线程（内核线程！！！）的调度和管理**, 变为**所有内核态其他守护线程的父线程**。

它的任务就是**管理和调度其他内核线程**kernel\_thread,会**循环执行**一个**kthread的函数**，该函数的作用就是运行**kthread\_create\_list全局链表**中维护kthread, 当我们调用**kernel\_thread创建的内核线程**会被**加入到此链表中**，因此**所有的内核线程**都是直接或者间接的以**kthreadd为父进程**

## 8.1 0号idle进程

在**smp系统**中，**每个处理器单元**有**独立的一个运行队列**，而**每个运行队列**上又**有一个idle进程**，即**有多少处理器单元**，就**有多少idle进程**。

**系统的空闲时间**，其实就是**指idle进程的"运行时间**"。

### 8.1.1 0号进程上下文信息--init\_task描述符

在内核初始化过程中，通过**静态定义**构造出了一个task\_struct接口，取名为**init\_task**变量，然后在**内核初始化的后期**，通过**rest\_init**()函数新建了**内核init线程，kthreadd内核线程**

所以**init\_task决定了系统所有进程、线程的基因, 它完成初始化后, 最终演变为0号进程idle, 并且运行在内核态**

在**init\_task进程执行后期**，它会**调用kernel\_thread**()函数创建**第一个核心进程kernel\_init**，同时init\_task进程**继续对Linux系统初始化**。在**完成初始化后**，**init\_task**会**退化为cpu\_idle进程**，当**Core 0**的**就绪队列**中**没有其它进程**时，该进程将会**获得CPU运行**。**新创建的1号进程kernel\_init将会逐个启动次CPU**,并**最终创建用户进程**！

备注：**core 0**上的**idle进程**由**init\_task进程退化**而来，而**AP的idle进程**则是**BSP在后面调用fork()函数逐个创建**的

内核在初始化过程中，当创建完init和kthreadd内核线程后，内核会发生**调度执行**，此时内核将使用该init\_task作为其task\_struct结构体描述符，当**系统无事可做**时，会**调度其执行**，此时**该内核会变为idle进程，让出CPU，自己进入睡眠，不停的循环**，查看**init\_task结构体**，其**comm字段为swapper**，作为idle进程的**描述符**。

init\_task描述符在init/init\_task.c中定义

```c
[init/init_task.c]
struct task_struct init_task = INIT_TASK(init_task);
EXPORT_SYMBOL(init_task);
```

### 8.1.2 进程堆栈init\_thread\_union

init\_task进程使用**init\_thread\_union**数据结构**描述的内存区域**作为**该进程的堆栈空间**，并且**和自身的thread\_info**参数**共用这一内存空间空间**

```c
#define INIT_TASK(tsk)	\
{									\
	.stack		= &init_thread_info,
}
```

**init\_thread\_info**则是一段**体系结构相关的定义**

```c
[arch/x86/include/asm/thread_info.h]
#define init_thread_info	(init_thread_union.thread_info)
#define init_stack		(init_thread_union.stack)
```

其中**init\_thread\_union**被定义在init/init\_task.c

```c
union thread_union init_thread_union __init_task_data =
        { INIT_THREAD_INFO(init_task) };
```

init\_task是用INIT\_THREAD\_INFO宏进行初始化的, 这个才是我们**真正体系结构相关的部分**

```c
[arch/x86/include/asm/thread_info.h]
#define INIT_THREAD_INFO(tsk)                   \
{                                               \
    .task           = &tsk,                 \
    .flags          = 0,                    \
    .cpu            = 0,                    \
    .addr_limit     = KERNEL_DS,            \
}
```

init\_thread\_info定义中**的\_\_init\_task\_data**表明该内核栈所在的区域**位于内核映像的init data区**，我们可以通过**编译完内核后**所产生的**System.map**来看到该变量及其对应的逻辑地址

### 8.1.3 进程内存空间

由于init\_task是一个**运行在内核空间的内核线程**,因此**其虚地址段mm为NULL**,但是必要时他还是**需要使用虚拟地址**的，因此**avtive\_mm被设置为init\_mm**

```c
.mm             = NULL,                                         \
.active_mm      = &init_mm,                                     \
```

```c
[mm/init-mm.c]
struct mm_struct init_mm = {
    .mm_rb          = RB_ROOT,
    .pgd            = swapper_pg_dir,
    .mm_users       = ATOMIC_INIT(2),
    .mm_count       = ATOMIC_INIT(1),
    .mmap_sem       = __RWSEM_INITIALIZER(init_mm.mmap_sem),
    .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
    .mmlist         = LIST_HEAD_INIT(init_mm.mmlist),
    INIT_MM_CONTEXT(init_mm)
};
```

### 8.1.4 0号进程的演化

#### 8.1.4.1 rest\_init创建init进程(PID=1)和kthread进程(PID=2)

在vmlinux的入口**startup\_32(head.S**)中为pid号为0的原始进程**设置了执行环境**，然后**原始进程开始执行start\_kernel**()完成Linux内核的初始化工作。包括初始化页表，初始化中断向量表，初始化系统时间等。

从**rest\_init**开始，Linux开始**产生进程**，因为init\_task是静态制造出来的，pid=0，它试图将**从最早的汇编代码**一直到**start\_kernel的执行**都纳入到**init\_task进程上下文**中。

这个**函数**其实是由**0号进程执行**的, 就是在这个函数中, 创建了**init进程**和**kthreadd**进程

start\_kernel**最后一个函数调用rest\_init**

```c
[init/main.c]
static noinline void __init_refok rest_init(void)
{
	int pid;

	rcu_scheduler_starting();
	smpboot_thread_init();

	kernel_thread(kernel_init, NULL, CLONE_FS);
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);

	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
}
```

1. 调用**kernel\_thread**()创建**1号内核线程**, 该线程**随后转向用户空间**, 演变为**init进程**

2. 调用**kernel\_thread**()创建**kthreadd内核线程**, pid=2。

3. init\_idle\_bootup\_task()：**当前0号进程**init\_task最终会**退化成idle进程**，所以这里调用**init\_idle\_bootup\_task**()函数，让**init\_task进程隶属到idle调度类**中。即选择idle的调度相关函数。

4. **调用schedule**()函数**切换当前进程**，在**调用该函数之前**，Linux系统中**只有两个进程**，即**0号进程init\_task**和**1号进程kernel\_init**，其中kernel\_init进程也是刚刚被创建的。**调用该函数后，1号进程kernel_init将会运行！！！**, 后续初始化都是使用该进程

5. 调用cpu\_idle()，0号线程进入idle函数的循环，在该循环中会周期性地检查。

##### 8.1.4.1.1 创建kernel\_init

产生第一个真正的进程(pid=1)

```c
kernel_thread(kernel_init, NULL, CLONE_FS);
```

##### 8.1.4.1.2 创建kthreadd

在rest\_init函数中，内核将通过下面的代码产生**第一个kthreadd(pid=2**)

```c
pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

#### 8.1.4.2 0号进程演变为idle


```c
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
```

因此我们回过头来看pid=0的进程，在**创建了init进程后**，pid=0的进程**调用cpu\_idle**()演变成了**idle进程**。

0号进程首先执行**init\_idle\_bootup\_task**, **让init\_task进程隶属到idle调度类中**。即选择idle的调度相关函数。

```c
void init_idle_bootup_task(struct task_struct *idle)
{
	idle->sched_class = &idle_sched_class;
}
```

接着通过schedule\_preempt\_disabled来**执行调用schedule()函数切换当前进程**，在调用该函数之前，Linux系统中只有两个进程，即**0号进程init\_task**和**1号进程kernel\_init**，其中kernel\_init进程也是刚刚被创建的。**调用该函数**后，**1号进程kernel\_init将会运行**

```c
void __sched schedule_preempt_disabled(void)
{
	sched_preempt_enable_no_resched();
	schedule();
	preempt_disable();
}
```

最后cpu\_startup\_entry**调用cpu\_idle\_loop()，0号线程进入idle函数的循环，在该循环中会周期性地检查**

```c
 void cpu_startup_entry(enum cpuhp_state state)
{
#ifdef CONFIG_X86
    boot_init_stack_canary();
#endif
    arch_cpu_idle_prepare();
    cpu_idle_loop();
}
```

其中cpu\_idle\_loop就是**idle进程的事件循环**，定义在kernel/sched/idle.c

整个过程简单的说就是，**原始进程(pid=0**)创建**init进程(pid=1**),然后演化成**idle进程(pid=0**)。**init进程**为**每个从处理器(运行队列**)创建出一个**idle进程(pid=0**)，然后**演化成/sbin/init**。

### 8.1.5 idle的运行与调度

#### 8.1.5.1 idle的workload--cpu\_idle\_loop

**idle**在系统**没有其他就绪的进程可执行**的时候才会**被调度**。不管是**主处理器**，还是**从处理器**，最后都是执行的**cpu\_idle\_loop**()函数

**idle进程**中并不执行什么有意义的任务，所以通常考虑的是两点

1. **节能**

2. **低退出延迟**。

```c
[kernel/sched/idle.c]
static void cpu_idle_loop(void)
{
	while (1) {
		__current_set_polling();
		quiet_vmstat();
		tick_nohz_idle_enter();

		while (!need_resched()) {
			check_pgt_cache();
			rmb();

			if (cpu_is_offline(smp_processor_id())) {
				rcu_cpu_notify(NULL, CPU_DYING_IDLE,
					       (void *)(long)smp_processor_id());
				smp_mb(); /* all activity before dead. */
				this_cpu_write(cpu_dead_idle, true);
				arch_cpu_idle_dead();
			}

			local_irq_disable();
			arch_cpu_idle_enter();

			if (cpu_idle_force_poll || tick_check_broadcast_expired())
				cpu_idle_poll();
			else
				cpuidle_idle_call();

			arch_cpu_idle_exit();
		}

		preempt_set_need_resched();
		tick_nohz_idle_exit();
		__current_clr_polling();

		smp_mb__after_atomic();

		sched_ttwu_pending();
		schedule_preempt_disabled();
	}
}
```

**循环判断need\_resched以降低退出延迟**，用**idle()来节能**。

**默认的idle实现是hlt指令**，hlt指令**使CPU处于暂停状态**，等待**硬件中断发生的时候恢复**，从而达到节能的目的。即**从处理器C0态变到C1态**(见**ACPI标准**)。这也是早些年windows平台上各种"处理器降温"工具的主要手段。当然idle也可以是在别的ACPI或者APM模块中定义的，甚至是自定义的一个idle(比如说nop)。

1. idle是一个进程，其pid为0。

2. **主处理器上的idle由原始进程(pid=0)演变**而来。**从处理器**上的idle由**init进程fork得到**，但是它们的**pid都为0**。

3. idle进程为**最低优先级**，且**不参与调度**，只是在**运行队列为空**的时候才被调度。

4. idle**循环等待need\_resched置位**。**默认使用hlt节能**。

#### 8.1.5.2 idle的运行时机

**idle进程优先级为MAX\_PRIO \- 20**。**早先版本**中，**idle是参与调度**的，所以**将其优先级设低点**，当**没有其他进程可以运行**时，才会调度执行idle。而**目前的版本**中idle并**不在运行队列中参与调度**，而是在**运行队列结构中含idle指针**，指向**idle进程**，在调度器发现**运行队列为空的时候运行**，调入运行

inux进程的**调度顺序**是按照**rt实时进程(rt调度器),normal普通进程(cfs调度器)，和idle**的顺序来调度的

那么可以试想**如果rt和cfs都没有可以运行**的任务，那么**idle才可以被调度**，那么他是通过**怎样的方式实现**的呢？

在**normal的调度类,cfs公平调度器**sched\_fair.c中

```c
static const struct sched_class fair_sched_class = {
    .next = &idle_sched_class,
```

也就是说，如果**系统中没有普通进程**，那么会**选择下个调度类优先级的进程**，即**使用idle\_sched\_class调度类进行调度的进程**

当**系统空闲**的时候，最后就是**调用idle的pick\_next\_task函数**，被定义在/kernel/sched/idle\_task.c中

```c
static struct task_struct *pick_next_task_idle(struct rq *rq)
{
        schedstat_inc(rq, sched_goidle);
        calc_load_account_idle(rq);
        return rq->idle;    //可以看到就是返回rq中idle进程。
}
```

这**idle进程**在**启动start\_kernel函数**的时候**调用init\_idle函数**的时候，把**当前进程（0号进程**）置为**每个rq运行队列的的idle**上。

```
rq->curr = rq->idle = idle;
```

这里idle就是调用start\_kernel函数的进程，就是**0号进程**。

## 8.2 1号init进程

### 8.2.1 执行函数kernel\_init()

0号进程创建1号进程的方式如下

```c
kernel_thread(kernel_init, NULL, CLONE_FS);
```

1号进程的**执行函数就是kernel\_init**, kernel\_init函数将完成**设备驱动程序的初始化**, 并调用**init\_post**函数启动**用户空间的init进程**。

```c
[init/main.c]
static int __ref kernel_init(void *unused)
{
	int ret;
    // 完成初始化工作，准备文件系统，准备模块信息
	kernel_init_freeable();
	/* need to finish all async __init code before freeing the memory */
	// 用以同步所有非同步函式呼叫的执行, 加速Linux Kernel开机的效率
	async_synchronize_full();
	free_initmem();
	mark_rodata_ro();
	// 设置运行状态SYSTEM_RUNNING
	system_state = SYSTEM_RUNNING;
	numa_default_policy();

	flush_delayed_fput();

	rcu_end_inkernel_boot();

	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d)\n",
		       ramdisk_execute_command, ret);
	}

	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}
	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;

	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/init.txt for guidance.");
}
```

| 执行流程 | 说明 |
| ------ |:-----|
| **kernel\_init\_freeable** | 调用kernel\_init\_freeable完成**初始化工作**，准备**文件系统**，准备**模块信息** |
| **async\_synchronize\_full** | 用以**同步所有非同步函式呼叫的执行**, 主要设计用来**加速Linux Kernel开机的效率**,避免在开机流程中等待硬体反应延迟,影响到开机完成的时间 |
| free\_initmem| 释放Linux Kernel介于\_\_init\_begin到 \_\_init\_end属于init Section的函数的所有内存.并会把Page个数加到变量totalram\_pages中 |
| system\_state | **设置运行状态SYSTEM_RUNNING** |
| 加载init进程，进入用户空间 | a,如果ramdisk\_execute\_command不为0,就执行该命令成为init User Process.<br>b,如果execute\_command不为0,就执行该命令成为init User Process.<br>c,如果上述都不成立,就依序执行如下指令<br>run\_init\_process(“/sbin/init”);<br>run\_init\_process(“/etc/init”);<br>run\_init\_process(“/bin/init”);<br>run\_init\_process(“/bin/sh”);<br>也就是说会按照顺序从/sbin/init, /etc/init, /bin/init 与 /bin/sh依序执行第一个 init User Process.<br> 如果都找不到可以执行的 init Process,就会进入Kernel Panic.如下所示panic(“No init found.  Try passing init= option to kernel. ”“See Linux Documentation/init.txt for guidance.”); |

由0号进程创建**1号进程（内核态**），**1号内核线程**负责执行**内核的部分初始化**工作及进行**系统配置(包括启动AP**)，并创建**若干**个用于**高速缓存**和**虚拟主存管理**的**内核线程**。

随后，**内核1号进程**就会**在/sbin, /etc, /bin寻找init程序**, 调用**do\_execve**()运行**可执行程序init**，并**演变成用户态1号进程**，即**init进程**。这个过程**并没有使用调用do\_fork**()，因此**两个进程都是1号进程**。

**该init程序会替换kernel\_init进程**（注意：并**不是创建一个新的进程来运行init程序**，而是一次变身，**使用sys\_execve函数**改变**核心进程的正文段**，将核心进程kernel\_init**转换成用户进程init**），此时处于内核态的**1号kernel\_init进程**将会转换为**用户空间内的1号进程init**。

init进程是**linux内核启动的第一个用户级进程**。init有许多很重要的任务，比如像**启动getty（用于用户登录）、实现运行级别、以及处理孤立进程**。

它**按照配置文件/etc/initab**的要求，完成**系统启动工作**，**创建编号为1号、2号...的若干终端注册进程getty**。

**每个getty进程**设置**其进程组标识号**，并**监视配置到系统终端的接口线路**。当检测到**来自终端的连接信号**时，**getty进程**将通过函数**do\_execve()执行注册程序login**，此时用户就可输入注册名和密码进入登录过程，如果成功，由**login程序**再通过函数**execv()执行/bin/shell**，该**shell进程**接收**getty进程的pid**，**取代原来的getty进程**。再由**shell直接或间接地产生其他进程**。

**用户进程init**将**根据/etc/inittab**中提供的信息**完成应用程序的初始化调用**。然后**init进程会执行/bin/sh产生shell界面**提供给用户来**与Linux系统进行交互**。

上述过程可描述为：**0号进程->1号内核进程->1号用户进程（init进程）->getty进程->shell进程**

在**系统完全起来之后**，init为**每个用户已退出的终端重启getty（这样下一个用户就可以登录**）。**init**同样也**收集孤立的进程**：当一个进程启动了一个子进程并且在子进程之前终止了，这个子进程立刻成为init的子进程。

### 8.2.2 关于init程序

init的最适当的位置（在Linux系统上）**是/sbin/init**。如果内核没有找到init，它就会**试着运行/bin/sh**，如果还是失败了，那么系统的启动就宣告失败了。

通过rpm \-qf查看系统程序所在的包, 目前系统的包是systemd.

## 8.3 2号kthreadd进程

```c
pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

**所有其它的内核线程的ppid都是2**，也就是说它们都是**由kthreadd thread创建**的

**所有的内核线程**在**大部分时间**里都**处于阻塞状态(TASK\_INTERRUPTIBLE**)只有在系统满足进程需要的某种资源的情况下才会运行

它的任务就是**管理和调度其他内核线程**kernel\_thread,会循环执行一个kthread的函数，该函数的作用就是运行kthread\_create\_list全局链表中维护的kthread,当我们调用kernel\_thread创建的内核线程会被加入到此链表中，因此所有的内核线程都是直接或者间接的以kthreadd为父进程

### 8.3.1 执行函数kthreadd()

```c
[kernel/kthread.c]
int kthreadd(void *unused)
{
    struct task_struct *tsk = current;

    /* Setup a clean context for our children to inherit. */
    set_task_comm(tsk, "kthreadd");
    ignore_signals(tsk);
    // 允许kthreadd在任意CPU上运行
    set_cpus_allowed_ptr(tsk, cpu_all_mask);
    set_mems_allowed(node_states[N_MEMORY]);

    current->flags |= PF_NOFREEZE;

    for (;;) {
 		// 首先将线程状态设置为TASK_INTERRUPTIBLE, 
 		// 如果当前没有要创建的线程则主动放弃CPU完成调度.此进程变为阻塞态
        set_current_state(TASK_INTERRUPTIBLE);
        // 没有需要创建的内核线程
        if (list_empty(&kthread_create_list))
            // 什么也不做, 执行一次调度, 让出CPU    
            schedule();					  

        // 运行到此表示kthreadd线程被唤醒(就是我们当前)
        // 设置进程运行状态为 TASK_RUNNING
        __set_current_state(TASK_RUNNING);
        //  加锁,
        spin_lock(&kthread_create_lock);
        while (!list_empty(&kthread_create_list)) {
            struct kthread_create_info *create;

			// 从链表中取得 kthread_create_info 
			// 结构的地址，在上文中已经完成插入操作(将
            // kthread_create_info 结构中的 list 
            // 成员加到链表中，此时根据成员 list 的偏移获得 create)
            create = list_entry(kthread_create_list.next,
                                struct kthread_create_info, list);

            /* 完成穿件后将其从链表中删除 */
            list_del_init(&create->list);

            /* 完成真正线程的创建 */
            spin_unlock(&kthread_create_lock);	
            create_kthread(create);
            spin_lock(&kthread_create_lock);
        }
        spin_unlock(&kthread_create_lock);
    }
    return 0;
}
```

kthreadd的核心是**for和while循环体**。

在**for循环**中，如果发现**kthread\_create\_list是一空链表(！！！**)，则**调用schedule调度函数**，因为此前已经将**该进程的状态设置为TASK\_INTERRUPTIBLE**，所以schedule的调用将会**使当前进程进入睡眠(会将进程从CPU运行队列中移除,可以通过显式的唤醒呼叫wakeup\_process()或需要处理的信号来唤醒它**)。

如果**kthread\_create\_list不为空**，则进入**while循环**，在该循环体中会**遍历该kthread\_create\_list列表**，对于该列表上的每一个entry，都会得到**对应的类型为struct kthread\_create\_info的节点的指针create**.

然后函数在kthread\_create\_list中**删除create对应的列表entry**，接下来**以create指针为参数调用create\_kthread(create**).

完成了**进程的创建**后**继续循环**，检查**kthread\_create\_list链表**，如果为空，则 **kthreadd 内核线程昏睡过去**

我们在内核中**通过kernel\_create或者其他方式创建一个内核线程**, 然后**kthreadd内核线程被唤醒**, 来**执行内核线程创建的真正工作**, 于是这里有**三个线程**

1. **kthreadd进程**已经光荣完成使命(接手**执行真正的创建工作**)，**睡眠**

2. **唤醒kthreadd的线程(！！！不是kthreadd线程本身！！！**)由于**新创建的线程**还**没有创建完毕而继续睡眠**(在**kthread\_create函数**中)

3. **新创建的线程**已经**正在运行kthread**()函数，但是由于还有其它工作没有做所以还没有最终创建完成.

#### 8.3.1.1 create\_kthread(struct kthread\_create\_info)完成内核线程创建

```c
[kernel/kthread.c]
static void create_kthread(struct kthread_create_info *create)
{
	int pid;

#ifdef CONFIG_NUMA
	current->pref_node_fork = create->node;
#endif
	/* We want our own signal handler (we take no signals by default). */
	// 其实就是调用首先构造一个假的上下文执行环境，
	// 最后调用do_fork()返回进程 id, 创建后的线程执行 kthread 函数
	pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
	if (pid < 0) {
		/* If user was SIGKILLed, I release the structure. */
		struct completion *done = xchg(&create->done, NULL);

		if (!done) {
			kfree(create);
			return;
		}
		create->result = ERR_PTR(pid);
		complete(done);
	}
}
```

里面会调用**kernel\_thread来生成一个新的进程**，**该进程的内核函数为kthread**，调用参数为

```
pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
```

创建的内核线程执行的函数是**kthread**()

### 8.3.2 新创建的内核线程执行函数kthread()

```c
static int kthread(void *_create)
{
    /* Copy data: it's on kthread's stack 
     create 指向 kthread_create_info 中的 kthread_create_info */
    struct kthread_create_info *create = _create;
    
     /* 新的线程创建完毕后执行的函数 */
    int (*threadfn)(void *data) = create->threadfn;
    /* 新的线程执行的参数  */
    void *data = create->data;
    struct completion *done;
    struct kthread self;
    int ret;

    self.flags = 0;
    self.data = data;
    init_completion(&self.exited);
    init_completion(&self.parked);
    current->vfork_done = &self.exited;

    /* If user was SIGKILLed, I release the structure. */
    done = xchg(&create->done, NULL);
    if (!done) {
            kfree(create);
            do_exit(-EINTR);
    }
    /* OK, tell user we're spawned, wait for stop or wakeup
     设置运行状态为 TASK_UNINTERRUPTIBLE  */
    __set_current_state(TASK_UNINTERRUPTIBLE);

     /*  current 表示当前新创建的 thread 的 task_struct 结构  */
    create->result = current;
    complete(done);
    /*  至此线程创建完毕, 执行任务切换，让出 CPU  */
    schedule();

    ret = -EINTR;

    if (!test_bit(KTHREAD_SHOULD_STOP, &self.flags)) {
            __kthread_parkme(&self);
            ret = threadfn(data);
    }
    /* we can't just return, we must preserve "self" on stack */
    do_exit(ret);
}
```

线程创建完毕:

1. **创建新thread的进程(原进程)恢复运行kthread\_create**()并且**返回新创建线程的任务描述符**

2. **新创建的线程**由于执行了**schedule**()调度，此时**并没有执行(！！！**).

直到我们手动使用**wake\_up\_process(p)唤醒新创建的线程**

3. **线程被唤醒**后, 会接着执行**threadfn(data)**

4. 得到**执行结果**, 将结果(整型类型)作为**参数**调用**do\_exit()函数**

### 8.3.3 小结

- **任何一个内核线程入口都是kthread**()

- 通过kthread\_create()创建的内核线程**不会立刻运行**．需要**手工wake up**

- 通过kthread\_create()创建的**内核线程**有**可能不会执行相应线程函数threadfn而直接退出(！！！**)

# 9 用户空间创建进程/线程的三种方法

| 系统调用 | 描述 |
|:-------------:|:-------------|
| fork | fork创造的**子进程是父进程的完整副本**，**复制了父亲进程的资源**，包括内存的task\_struct内容 |
| vfork | vfork创建的**子进程与父进程共享数据段(数据段！！！**),而且由vfork()创建的**子进程将先于父进程运行** |
| clone | Linux上**创建线程**一般使用的是**pthread库**.实际上linux也给我们提供了**创建线程的系统调用，就是clone** |

**fork, vfork和clone**的**系统调用**的**入口地址**分别是**sys\_fork(),sys\_vfork()和sys\_clone**(), 而他们的**定义是依赖于体系结构**的, 因为在**用户空间和内核空间之间传递参数的方法因体系结构而异**

## 9.1 系统调用的参数传递

由于**系统调用**是**通过中断进程从用户态到内核态**的一种特殊的函数调用，**没有用户态或者内核态的堆栈(！！！**)可以被用来在调用函数和被调函数之间进行**参数传递**。

**系统调用通过CPU的寄存器来进行参数传递**。在进行**系统调用之前**，系统调用的**参数被写入CPU的寄存器**，而在**实际调用系统服务例程之前**，内核**将CPU寄存器的内容拷贝到内核堆栈**中，实现参数的传递。

上面**函数的任务**就是**从处理器的寄存器**中**提取用户空间提供的信息**,并调用**体系结构无关的\_do\_fork（或者早期的do\_fork）函数**,负责**进程的复制**

Linux有一个**TLS(Thread Local Storage)机制**,**clone**的标识**CLONE\_SETTLS**接受一个参数来**设置线程的本地存储区**。

sys\_clone也因此**增加了一个int参数**来传入相应的tls\_val。**sys\_clone通过do\_fork**来调用copy\_process完成进程的复制，它调用特定的copy\_thread和copy\_thread把相应的系统调用参数从**pt\_regs寄存器列表中提取出来**，这个参数仍然是体系结构相关的.

所以Linux引入一个新的**CONFIG\_HAVE\_COPY\_THREAD\_TLS**，和一个新的**COPY\_THREAD\_TLS**接受TLS参数为额外的长整型（系统调用参数大小）的争论。改变sys\_clone的TLS参数unsigned long，并传递到**copy\_thread\_tls**。

```c
[include/linux/sched.h]
extern long _do_fork(unsigned long, unsigned long, unsigned long, int __user *, int __user *, unsigned long);
extern long do_fork(unsigned long, unsigned long, unsigned long, int __user *, int __user *);

#ifndef CONFIG_HAVE_COPY_THREAD_TLS
long do_fork(unsigned long clone_flags,
              unsigned long stack_start,
              unsigned long stack_size,
              int __user *parent_tidptr,
              int __user *child_tidptr)
{
    return _do_fork(clone_flags, stack_start, stack_size,
                        parent_tidptr, child_tidptr, 0);
}
#endif
```

**新版本**的系统中**clone的TLS设置标识**会通过**TLS参数传递**,**因此\_do\_fork替代了老版本的do\_fork**。

**老版本的do\_fork**只有在如下情况才会定义

- 只有当系统**不支持通过TLS参数**传递而是**使用pt\_regs寄存器列表**传递时

- **未定义CONFIG\_HAVE\_COPY\_THREAD\_TLS宏**

| 参数 | 描述 |
| ------------- |:-------------|
| clone\_flags | 与clone()参数flags相同,用来控制进程复制过的一些属性信息,描述你需要**从父进程继承哪些资源**。该**标志位的4个字节**分为**两部分**。**最低的一个字节**为**子进程结束**时**发送给父进程的信号代码**，通常为**SIGCHLD**；**剩余的三个字节**则是**各种clone标志的组合**（本文所涉及的标志含义详见下表），也就是若干个标志之间的或运算。通过clone标志可以有选择的对父进程的资源进行复制； |
| stack\_start | 与clone()参数stack\_start相同, **子进程用户态(！！！)堆栈的地址** |
| regs | 是一个指向了**寄存器集合的指针**,其中以原始形式,**保存了调用的参数**,该**参数使用的数据类型**是**特定体系结构的struct pt\_regs**，其中**按照系统调用执行时寄存器在内核栈上的存储顺序**,保存了**所有的寄存器**,即**指向内核态堆栈通用寄存器值的指针**，**通用寄存器的值**是在从**用户态切换到内核态时被保存到内核态堆栈中的**(指向**pt\_regs**结构体的指针。当系统发生**系统调用**，即**用户进程从用户态切换到内核态**时，该结构体**保存通用寄存器中的值**，并**被存放于内核态的堆栈**中) |
| stack\_size | **用户状态下栈的大小**, 该参数通常是不必要的, **总被设置为0** |
| parent\_tidptr | 与clone的ptid参数相同,**父进程在用户态下pid的地址**，该参数在**CLONE\_PARENT\_SETTID标志被设定时有意义** |
| child\_tidptr | 与clone的ctid参数相同,**子进程在用户态下pid的地址**，该参数在**CLONE\_CHILD\_SETTID标志被设定时有意义** |

clone\_flags如下表所示

![CLONE_FLAGS](./images/7.jpg)

## 9.2 sys\_fork的实现

```c
早期实现
asmlinkage long sys_fork(struct pt_regs regs)
{
    return do_fork(SIGCHLD, regs.rsp, &regs, 0);
}

新版本实现
#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
        return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
#else
        return -EINVAL;
#endif
}
#endif
```

- **唯一使用的标志是SIGCHLD**。这意味着在**子进程终止**后将**发送信号SIGCHLD**信号**通知父进程**

- 写时复制(COW)技术, **最初父子进程的栈地址相同**, 但是如果**操作栈地址并写入数据**, 则COW机制会为**每个进程**分别创建一个**新的栈副本**

- 如果**do\_fork成功**, 则**新建进程的pid作为系统调用的结果返回**, 否则**返回错误码**

## 9.3 sys\_vfork的实现

```c
早期实现
asmlinkage long sys_vfork(struct pt_regs regs)
{
    return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, regs.rsp, &regs, 0);
}

新实现
#ifdef __ARCH_WANT_SYS_VFORK
SYSCALL_DEFINE0(vfork)
{
        return _do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
                        0, NULL, NULL, 0);
}
#endif
```

相较于sys\_vfork, 多使用了**额外的标志CLONE\_VFORK | CLONE\_VM**

## 9.4 sys\_clone的实现

```c
早期实现
casmlinkage int sys_clone(struct pt_regs regs)
{
    /* 注释中是i385下增加的代码, 其他体系结构无此定义
    unsigned long clone_flags;
    unsigned long newsp;

    clone_flags = regs.ebx;
    newsp = regs.ecx;*/
    if (!newsp)
        newsp = regs.esp;
    return do_fork(clone_flags, newsp, &regs, 0);
}

新版本
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
                 int __user *, parent_tidptr,
                 unsigned long, tls,
                 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
                 int __user *, parent_tidptr,
                 int __user *, child_tidptr,
                 unsigned long, tls)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
                int, stack_size,
                int __user *, parent_tidptr,
                int __user *, child_tidptr,
                unsigned long, tls)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
                 int __user *, parent_tidptr,
                 int __user *, child_tidptr,
                 unsigned long, tls)
#endif
{
        return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
#endif
```

**sys\_clone的标识不再是硬编码**的,而是通过**各个寄存器参数传递到系统调用**, **clone也不再复制进程的栈**,而是**可以指定新的栈地址**,在**生成线程时,可能需要这样做**,线程可能与父进程共享地址空间，但是**线程自身的栈可能在另外一个地址空间**

另外还指令了用户空间的两个指针(parent\_tidptr和child\_tidptr), 用于**与线程库通信**

# 10 创建子进程流程

\_**do\_fork**和**do\_fork**在**进程的复制**的时候并没有太大的区别,他们就只是在**进程tls复制**的过程中实现有**细微差别**

## 10.1 \_do\_fork的流程

**所有进程复制(创建)的fork机制**最终都调用了**kernel/fork.c中的\_do\_fork**(一个体系结构无关的函数)

```c
long _do_fork(unsigned long clone_flags,
      unsigned long stack_start,
      unsigned long stack_size,
      int __user *parent_tidptr,
      int __user *child_tidptr,
      unsigned long tls)
{
    struct task_struct *p;
    int trace = 0;
    long nr;
  
    if (!(clone_flags & CLONE_UNTRACED)) {
    if (clone_flags & CLONE_VFORK)
        trace = PTRACE_EVENT_VFORK;
    else if ((clone_flags & CSIGNAL) != SIGCHLD)
        trace = PTRACE_EVENT_CLONE;
    else
        trace = PTRACE_EVENT_FORK;
  
    if (likely(!ptrace_event_enabled(current, trace)))
        trace = 0;
    }
  	/* 复制进程描述符，copy_process()的返回值是一个 task_struct 指针 */
    p = copy_process(clone_flags, stack_start, stack_size,
         child_tidptr, NULL, trace, tls);

    if (!IS_ERR(p)) {
    struct completion vfork;
    struct pid *pid;
  
    trace_sched_process_fork(current, p);
  	/*  得到新创建的进程的pid信息  */
    pid = get_task_pid(p, PIDTYPE_PID);
    nr = pid_vnr(pid);
  
    if (clone_flags & CLONE_PARENT_SETTID)
        put_user(nr, parent_tidptr);
  	
    /* 如果调用的 vfork()方法，初始化 vfork 完成处理信息 */
    if (clone_flags & CLONE_VFORK) {
        p->vfork_done = &vfork;
        init_completion(&vfork);
        get_task_struct(p);
    }
	/*  将子进程加入到调度器中，为其分配 CPU，准备执行  */
    wake_up_new_task(p);
  
    /* forking complete and child started to run, tell ptracer */
    if (unlikely(trace))
        ptrace_event_pid(trace, pid);
  	
    /*  如果是 vfork，将父进程加入至等待队列，等待子进程完成  */
    if (clone_flags & CLONE_VFORK) {
        if (!wait_for_vfork_done(p, &vfork))
        ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
    }
  
    put_pid(pid);
    } else {
    nr = PTR_ERR(p);
    }
    return nr;
}
```

1. 调用**copy\_process**为子进程**复制出一份进程信息**

2. 如果是**vfork**（设置了**CLONE\_VFORK和ptrace标志**）初始化完成处理信息

3. 调用**wake\_up\_new\_task**()将**子进程加入调度器**，为之分配CPU. 计算此进程的**优先级和其他调度参数**，将**新的进程加入到进程调度队列并设此进程为可被调度的**，以后**这个进程**可以被**进程调度模块**调度执行。

4. 如果是**vfork**，**父进程等待子进程完成exec替换自己的地址空间**

## 10.2 copy\_process流程

fork的大部分事情，它主要完成讲**父进程的运行环境复制到新的子进程**，比如信号处理、文件描述符和进程的代码数据等。

1. **dup\_task\_struct**()复制当前的task\_struct, 并为其分配了**新的堆栈**.分配一个**新的进程控制块**，包括**新进程在kernel中的堆栈**。新的进程控制块会复制父进程的进程控制块，但是**因为每个进程都有一个kernel堆栈**，新进程的堆栈将被设置成**新分配的堆栈**

2. 检查**进程数是否超过限制**,两个因素:操作系统和内存大小

3. 初始化**自旋锁**、**挂起信号**、**CPU定时器**等

4. 调用**sched\_fork**()初始化**进程数据结构**，并把进程状态设置为**TASK\_RUNNING**. 设置子进程调度相关的参数，即子进程的运行CPU、初始时间片长度和静态优先级等

5. copy\_semundo()复制父进程的**semaphore undo\_list**到子进程

6. copy\_files()、copy\_fs()复制父进程**文件系统相关**的环境到子进程

7. copy\_sighand()、copy\_signal()复制父进程**信号处理相关**的环境到子进程

8. copy\_mm()复制父进程**内存管理相关**的环境到子进程，包括**页表、地址空间和代码数据**

9. copy\_thread\_tls()中将**父进程的寄存器上下文复制给子进程**，保证了**父子进程的堆栈信息是一致**的. 设置子进程的执行环境，如子进程运行时各CPU寄存器的值、子进程的**kernel栈**的起始地址

10. 将**ret\_from\_fork的地址**设置为**eip寄存器的值**

11. 为**新进程分配并设置新的pid**

12. 将子进程加入到全局的进程队列中

13. 设置子进程的进程组ID和对话期ID等

14. 最终**子进程**从**ret\_from\_fork！！！**开始执行

简单的说，copy\_process()就是将父进程的运行环境复制到子进程并对某些子进程特定的环境做相应的调整。

### 10.2.1 dup\_task\_struct()产生新的task\_struct

1. 调用**alloc\_task\_struct\_node**分配一个**task\_struct节点**

2. 调用**alloc\_thread\_info\_node**分配一个**thread\_info节点**，其实是**分配了一个thread\_union联合体**,将**栈底返回给ti**

```c
union thread_union {
   struct thread_info thread_info;
  unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

- 最后将**栈底的值ti**赋值给**新节点的栈**

- 最终执行完**dup\_task\_struct**之后，**子进程除了tsk->stack指针不同**之外，**全部都一样**！

### 10.2.2 sched\_fork()流程

```c
int sched_fork(unsigned long clone_flags, struct task_struct *p)
{
	unsigned long flags;
	int cpu = get_cpu();

	__sched_fork(clone_flags, p);

	//  将子进程状态设置为 TASK_RUNNING
	p->state = TASK_RUNNING;
	
	p->prio = current->normal_prio;

    if (unlikely(p->sched_reset_on_fork)) {
		if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
			p->policy = SCHED_NORMAL;
			p->static_prio = NICE_TO_PRIO(0);
			p->rt_priority = 0;
		} else if (PRIO_TO_NICE(p->static_prio) < 0)
			p->static_prio = NICE_TO_PRIO(0);

		p->prio = p->normal_prio = __normal_prio(p);
		set_load_weight(p);

		p->sched_reset_on_fork = 0;
	}

	if (dl_prio(p->prio)) {
		put_cpu();
		return -EAGAIN;
	} else if (rt_prio(p->prio)) {
		p->sched_class = &rt_sched_class;
	} else {
		p->sched_class = &fair_sched_class;
	}

	//  ……

	//  为子进程分配 CPU
	set_task_cpu(p, cpu);

	put_cpu();
	return 0;
}
```

我们可以看到sched\_fork大致完成了两项重要工作，

- 一是将子进程状态设置为TASK\_RUNNING, 并设置调度相关字段

- 二是**为其分配CPU**

### 10.2.3 copy\_thread和copy\_thread\_tls流程

如果**未定义CONFIG\_HAVE\_COPY\_THREAD\_TLS宏**默认则使用copy\_thread同时将定义copy\_thread\_tls为copy\_thread

**单独这个函数**是因为**这个复制操作与其他操作都不相同**,这是一个**特定于体系结构的函数**，用于**复制进程中特定于线程(thread\-special)的数据**,重要的就是填充**task\_struct->thread的各个成员**，这是一个thread\_struct类型的结构, 其**定义是依赖于体系结构**的。它包含了**所有寄存器(和其他信息！！！所有寄存器信息在thread里面！！！**)，内核在进程之间**切换时需要保存和恢复的进程的信息**。

该函数用于**设置子进程的执行环境**，如子进程运行时**各CPU寄存器的值**、子进程的**内核栈的起始地址**（**指向内核栈的指针通常也是保存在一个特别保留的寄存器**中）

32位架构的copy\_thread\_tls函数

```c
[arch/x86/kernel/process_32.c]
int copy_thread_tls(unsigned long clone_flags, unsigned long sp,
    unsigned long arg, struct task_struct *p, unsigned long tls)
{
    struct pt_regs *childregs = task_pt_regs(p);
    struct task_struct *tsk;
    int err;
	/* 获取寄存器的信息 */
    p->thread.sp = (unsigned long) childregs;
    p->thread.sp0 = (unsigned long) (childregs+1);
    memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps));

    if (unlikely(p->flags & PF_KTHREAD)) {
        /* kernel thread 内核线程的设置  */
        memset(childregs, 0, sizeof(struct pt_regs));
        p->thread.ip = (unsigned long) ret_from_kernel_thread;
        task_user_gs(p) = __KERNEL_STACK_CANARY;
        childregs->ds = __USER_DS;
        childregs->es = __USER_DS;
        childregs->fs = __KERNEL_PERCPU;
        childregs->bx = sp;     /* function */
        childregs->bp = arg;
        childregs->orig_ax = -1;
        childregs->cs = __KERNEL_CS | get_kernel_rpl();
        childregs->flags = X86_EFLAGS_IF | X86_EFLAGS_FIXED;
        p->thread.io_bitmap_ptr = NULL;
        return 0;
    }
    /* 将当前寄存器信息复制给子进程 */
    *childregs = *current_pt_regs();
    /* 子进程 eax 置 0，因此fork 在子进程返回0 */
    childregs->ax = 0;
    if (sp)
        childregs->sp = sp;
	/* 子进程ip设置为ret_from_fork，因此子进程从ret_from_fork开始执行  */
    p->thread.ip = (unsigned long) ret_from_fork;
    task_user_gs(p) = get_user_gs(current_pt_regs());

    p->thread.io_bitmap_ptr = NULL;
    tsk = current;
    err = -ENOMEM;

    if (unlikely(test_tsk_thread_flag(tsk, TIF_IO_BITMAP))) {
        p->thread.io_bitmap_ptr = kmemdup(tsk->thread.io_bitmap_ptr,
                        IO_BITMAP_BYTES, GFP_KERNEL);
        if (!p->thread.io_bitmap_ptr) {
            p->thread.io_bitmap_max = 0;
            return -ENOMEM;
        }
        set_tsk_thread_flag(p, TIF_IO_BITMAP);
    }

    err = 0;

    /* 为进程设置一个新的TLS */
    if (clone_flags & CLONE_SETTLS)
        err = do_set_thread_area(p, -1,
            (struct user_desc __user *)tls, 0);

    if (err && p->thread.io_bitmap_ptr) {
        kfree(p->thread.io_bitmap_ptr);
        p->thread.io_bitmap_max = 0;
    }
    return err;
}
```

这里解释了两个相当重要的问题！

一是，**为什么fork在子进程中返回0！！！**，原因是childregs\-\>ax = 0;这段代码**将子进程的 eax 赋值为0！！！**

二是，**p->thread.ip = (unsigned long) ret\_from\_fork**;将子进程的 ip 设置为 ret\_form\_fork 的**首地址**，因此**子进程是从ret\_from\_fork 开始执行的！！！**

# 11 用户程序结束进程

**应用程序**使用**系统调用exit**()来结束一个进程，此**系统调用接受一个退出原因代码**，**父进程**可以**使用wait()系统调用**来**获取此退出代码**，从而**知道子进程退出的原因**。

对应到kernel，此系统调用**sys\_exit\_group**()，它的基本流程如下：

1. 将**信号SIGKILL**加入到其他线程的**信号队列**中，并唤醒这些线程。

2. 此线程执行do\_exit()来退出。

do\_exit()完成线程退出的任务，其主要功能是将线程占用的系统资源释放，do\_exit()的基本流程如下： 

1. 将进程内存管理相关的资源释放

2. 将进程ICP semaphore相关资源释放

3. \_\_exit\_files()、\_\_exit\_fs()将进程文件管理相关的资源释放。

4. exit\_thread()只要目的是释放平台相关的一些资源。

5. exit\_notify()在Linux中进程退出时要将其退出的原因告诉父进程，父进程调用wait()系统调用后会在一个等待队列上睡眠。

6. schedule()调用进程调度器，因为此进程已经退出，切换到其他进程。

# 12 进程状态变化过程

进程的创建到执行过程如下图所示

![进程的状态](./images/8.jpg)

# 13 内核线程

**内核线程**就是**内核的分身**，一个分身可以处理一件特定事情。**内核线程的调度由内核负责**，一个**内核线程**处于**阻塞状态**时**不影响其他的内核线程**，因为其是调度的基本单位。

**内核线程只运行在内核态**

因此，它**只能使用大于PAGE\_OFFSET**（**传统的x86\_32上是3G**）的**地址空间(！！！**)。

## 13.1 概述

**内核线程**是直接由**内核本身启动的进程**。**内核线程**实际上是将**内核函数**委托给**独立的进程**，它与内核中的其他进程"并行"执行。内核线程经常被称之为**内核守护进程**。

他们执行下列**任务**

- **周期性**地将**修改的内存页与页来源块设备同步**

- 如果**内存页很少使用**，则**写入交换区**

- 管理**延时动作**,　如**２号进程接手内核进程的创建**

- 实现**文件系统的事务日志**

**内核线程**主要有**两种类型**

1. 线程启动后一直**等待**，直至**内核请求线程执行某一特定操作**。

2. 线程启动后按**周期性间隔运行**，检测特定资源的使用，在用量超出或低于预置的限制时采取行动。

**内核线程**由**内核自身生成**，其**特点**在于

1. 它们在**CPU的管态执行**，而**不是用户态**。

2. 它们只可以访问**虚拟地址空间的内核部分**（**高于TASK\_SIZE的所有地址**），但**不能访问用户空间**

**内核线程**和**普通的进程**间的区别在于**内核线程没有独立的地址空间**，**mm指针被设置为NULL**；它**只在内核空间运行**，从来**不切换到用户空间**去；并且和普通进程一样，可以**被调度**，也可以**被抢占**。

## 13.2 内核线程的创建

### 13.2.1 创建内核线程接口

内核线程可以通过两种方式实现：

- **kernel\_thread和daemonize**

将**一个函数**传递给**kernel\_thread**创建并初始化一个task，**该函数**接下来负责帮助内核**调用daemonize**已转换为**内核守护进程**

- **kthead\_create和kthread\_run**

创建内核更常用的方法是辅助函数kthread\_create，该函数创建一个新的内核线程。**最初线程是停止的**，需要使用**wake\_up\_process**启动它。

使用**kthread\_run**，与kthread\_create不同的是，其创建新线程后**立即唤醒它**，其本质就是先用**kthread\_create**创建一个内核线程，然后通过wake\_up\_process唤醒它

### 13.2.2 2号进程kthreadd

见前面2号进程的创建. 

参见kthreadd函数, 它会**循环的是查询工作链表**static LIST\_HEAD(kthread\_create\_list)中是否有需要被创建的内核线程

我们的通过**kthread\_create**执行的操作, 只是在内核线程任务队列**kthread\_create\_list**中增加了一个**create任务**, 然后会**唤醒kthreadd进程来执行真正的创建操作**

**内核线程**会出现在**系统进程列表中**,但是在**ps的输出**中**进程名command由方括号包围**, 以便**与普通进程区分**。

如下图所示, 我们可以看到系统中,**所有内核线程都用[]标识**,而且这些进程**父进程id均是2**, 而**2号进程kthreadd**的**父进程是0号进程**

![config](./images/9.jpg)

### 13.2.3 kernel\_thread()创建内核线程

kernel\_thread()的实现**经历过很多变革**

**早期**的kernel\_thread()执行**更底层的操作**, 直接创建了**task\_struct**并进行**初始化**,

引入了kthread\_create和kthreadd 2号进程后, **kernel\_thread**()的实现也由**统一的\_do\_fork**(或者早期的do\_fork)托管实现

# 14 可执行程序的加载和运行

**fork, vfork**等复制出来的进程是**父进程的一个副本**, 那么如何我们想**加载新的程序**, 可以通过**execve系统调用**来加载和启动新的程序。

## 14.1 exec()函数族

exec函数一共有六个，其中**execve**为**内核级系统调用**，其他（**execl**，**execle**，**execlp**，**execv**，**execvp**）都是调用**execve**的库函数。

```c
int execl(const char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg,
                  ..., char * const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
```

## 14.2 可执行程序相关数据结构

Linux下标准的可执行文件格式是ELF. ELF(**Executable and Linking Format**)是一种**对象文件的格式**, 作为**缺省的二进制文件格式**来使用. 

linux也支持**其他不同的可执行程序格式**, 各个**可执行程序的执行方式不尽相同**, 因此linux内核**每种被注册的可执行程序格式**都用**linux\_bin\_fmt**来存储, 其中记录了**可执行程序的加载和执行函数**

同时我们需要一种方法来**保存可执行程序的信息**,比如可执行文件的路径,运行的参数和环境变量等信息，即**linux\_binprm结构**

### 14.2.1 struct linux\_binprm结构描述一个可执行程序的信息

struct linux\_binprm保存**要执行的文件相关的信息**, 包括可执行程序的路径, 参数和环境变量的信息

```c
[include/linux/binfmts.h]
struct linux_binprm {
    char buf[BINPRM_BUF_SIZE];	// 保存可执行文件的头128字节
#ifdef CONFIG_MMU
    struct vm_area_struct *vma;
    unsigned long vma_pages;
#else
# define MAX_ARG_PAGES  32
    struct page *page[MAX_ARG_PAGES];
#endif
    struct mm_struct *mm;
    /* current top of mem , 当前内存页最高地址*/
    unsigned long p; 
    unsigned int
            cred_prepared:1,
            cap_effective:1;
#ifdef __alpha__
    unsigned int taso:1;
#endif
    unsigned int recursion_depth; 
    /* 要执行的文件 */
    struct file * file;	 
    struct cred *cred;      /* new credentials */
    int unsafe; 
    unsigned int per_clear;
    /* 命令行参数和环境变量数目 */
    int argc, envc;		
    // 要执行的文件的名称 
    const char * filename; 
    //  要执行的文件的真实名称，通常和filename相同
    const char * interp; 
    unsigned interp_flags;
    unsigned interp_data;
    unsigned long loader, exec;
};
```

### 14.2.2 struct linux\_binfmt可执行格式的结构

linux内核对所支持的**每种可执行的程序类型(！！！**)都有个**struct linux\_binfmt的数据结构**，定义如下

```c
[include/linux/binfmts.h]
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);
    int (*load_shlib)(struct file *);
    int (*core_dump)(struct coredump_params *cprm);
    unsigned long min_coredump;     /* minimal dump size */
 };
```

其提供了**3种方法来加载和执行可执行程序**

- **load\_binary**

通过读**存放在可执行文件中的信息**为当前进程**建立一个新的执行环境**

- **load\_shlib**

用于**动态的把一个共享库捆绑到一个已经在运行的进程**, 这是由**uselib()系统调用激活**的

- **core\_dump**

在名为core的文件中, 存放**当前进程的执行上下文**.这个文件通常是在**进程**接收到一个缺省操作为"**dump**"的**信号**时被创建的, 其**格式**取决于**被执行程序的可执行类型**

**所有的linux\_binfmt对象**都处于**一个链表**中,第一个元素的地址存放在**formats变量！！！**中,可以通过**调用register\_binfmt()和unregister\_binfmt()函数**在链表中**插入和删除元素**,在**系统启动期间**,为**每个编译进内核的可执行格式**都执行**registre\_fmt()函数**.当实现了一个新的可执行格式的模块正被装载时,也执行这个函数,当**模块被卸载**时, 执行**unregister\_binfmt()函数**.

当我们**执行一个可执行程序**的时候,内核会list\_for\_each\_entry**遍历所有注册的linux\_binfmt对象**,对其**调用load\_binrary方法来尝试加载**, 直到加载成功为止.

## 14.3 execve加载可执行程序的过程

内核中实际执行**execv()或execve()系统调用**的程序是**do\_execve**()

这个函数**先打开目标映像文件**，

并**从目标文件的头部**（**第一个字节**开始）读入若干（当前Linux内核中是**128）字节**（实际上就是**填充ELF文件头**，下面的分析可以看到），

然后**调用另一个函数search\_binary\_handler**()，在此函数里面，它会**搜索我们上面提到的Linux支持的可执行文件类型队列**，让各种可执行程序的处理程序前来认领和处理。

如果**类型匹配**，则**调用load\_binary函数**指针所指向的处理函数来**处理目标映像文件**。在**ELF文件格式**中，处理函数是**load\_elf\_binary函数**：

sys\_execve() > do\_execve() > do\_execveat\_common > search\_binary\_handler() > load\_elf\_binary()

