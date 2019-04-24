
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Linux中3个特殊的进程](#1-linux中3个特殊的进程)
* [2 idle的创建](#2-idle的创建)
	* [2.1 0号进程上下文信息--init\_task描述符](#21-0号进程上下文信息-init_task描述符)
		* [2.1.1 进程堆栈init\_thread\_union](#211-进程堆栈init_thread_union)
		* [2.1.2 进程内存空间](#212-进程内存空间)
* [3 0号进程的演化](#3-0号进程的演化)
	* [3.1 rest\_init创建init进程(PID=1)和kthread进程(PID=2)](#31-rest_init创建init进程pid1和kthread进程pid2)
		* [3.1.1 创建kernel\_init](#311-创建kernel_init)
		* [3.1.2 创建kthreadd](#312-创建kthreadd)
	* [3.2 0号进程演变为idle](#32-0号进程演变为idle)
* [4 idle的运行与调度](#4-idle的运行与调度)
	* [4.1 idle的workload--cpu\_idle\_loop](#41-idle的workload-cpu_idle_loop)
	* [4.2 idle的调度和运行时机](#42-idle的调度和运行时机)
* [5 idle进程总结](#5-idle进程总结)
* [6 附录\-\-rest\_init的执行解析](#6-附录-rest_init的执行解析)

<!-- /code_chunk_output -->

# 1 Linux中3个特殊的进程

Linux下有**3个特殊的进程**，**idle**进程($**PID = 0**$), init进程($**PID = 1**$)和**kthreadd**($**PID = 2**$)

- idle进程由**系统自动创建**, 运行在**内核态**

idle进程其pid=0，其前身是**系统创建的第一个进程**，也是**唯一一个没有通过fork或者kernel\_thread产生的进程**。完成加载系统后，演变为**进程调度、交换**

- **init**进程由**idle**通过**kernel\_thread创建**，在**内核空间（！！！）完成初始化后**, 加载**init程序**, 并最终**用户空间**

由**0进程创建**，完成**系统的初始化**.是系统中**所有其它用户进程（！！！）的祖先进程**.Linux中的**所有进程**都是有**init进程创建并运行**的。首先Linux内核启动，然后**在用户空间中启动init进程**，再**启动其他系统程**。在**系统启动完成后**，**init**将变为**守护进程监视系统其他进程**

- kthreadd进程由**idle**通过**kernel\_thread**创建,并始终运行在**内核空间**,负责所有**内核线程（内核线程！！！）的调度和管理**

它的任务就是**管理和调度其他内核线程**kernel\_thread,会**循环执行**一个**kthread的函数**，该函数的作用就是运行**kthread\_create\_list全局链表**中维护kthread, 当我们调用**kernel\_thread创建的内核线程**会被**加入到此链表中**，因此**kthreadd为父进程**

我们下面就详解分析0号进程的前世(init\_task)今生(idle)

# 2 idle的创建

在**smp系统**中，**每个处理器单元**有**独立的一个运行队列**，而**每个运行队列**上又**有一个idle进程**，即**有多少处理器单元**，就**有多少idle进程**。

idle进程其pid=0，其前身是系统创建的第一个进程，也是唯一一个没有通过fork()产生的进程。在smp系统中，每个处理器单元有独立的一个运行队列，而每个运行队列上又有一个idle进程，即有多少处理器单元，就有多少idle进程。**系统的空闲时间**，其实就是**指idle进程的"运行时间**"。既然是idle是进程，那我们来看看idle是如何被创建，又具体做了哪些事情？

我们知道系统是从BIOS加电自检，载入MBR中的引导程序(LILO/GRUB),再加载linux内核开始运行的，一直到指定shell开始运行告一段落，这时用户开始操作Linux。

## 2.1 0号进程上下文信息--init\_task描述符

**init\_task**是内核中所有进程、线程的task\_struct雏形，在内核初始化过程中，通过**静态定义**构造出了一个task\_struct接口，取名为**init\_task**，然后在**内核初始化的后期**，通过**rest\_init**()函数新建了**内核init线程，kthreadd内核线程**

- **内核init线程**，最终**执行/sbin/init进程**，变为**所有用户态程序的根进程（pstree命令显示**）,即**用户空间的init进程**

**开始的init**是有kthread\_thread创建的**内核线程**,他在**完成初始化工作后**,转向**用户空间**, 并且生成所有用户进程的祖先

- **内核kthreadd内核线程**，变为**所有内核态其他守护线程的父线程**。

它的任务就是**管理和调度其他内核线程kernel\_thread**,会循环执行一个kthread的函数，该函数的作用就是运行kthread\_create\_list全局链表中维护的kthread,当我们调用kernel\_thread创建的内核线程会被加入到此链表中，因此**所有的内核线程**都是直接或者间接的以kthreadd为父进程

![pa-aux](./images/ps-aux.jpg)

所以**init\_task决定了系统所有进程、线程的基因,它完成初始化后,最终演变为0号进程idle,并且运行在内核态**

内核在初始化过程中，当创建完init和kthreadd内核线程后，内核会发生**调度执行**，此时内核将使用该init\_task作为其task\_struct结构体描述符，当**系统无事可做**时，会**调度其执行**，此时**该内核会变为idle进程，让出CPU，自己进入睡眠，不停的循环**，查看**init\_task结构体**，其**comm字段为swapper**，作为idle进程的**描述符**。

>idle的运行时机
>
>**idle进程优先级为MAX\_PRIO \- 20**。**早先版本**中，**idle是参与调度**的，所以**将其优先级设低点**，当**没有其他进程可以运行**时，才会调度执行idle。而**目前的版本**中idle并**不在运行队列中参与调度**，而是在**运行队列结构中含idle指针**，指向**idle进程**，在调度器发现**运行队列为空的时候运行**，调入运行

简言之, **内核中init\_task变量就是是进程0使用的进程描述符**，也是Linux系统中第一个进程描述符，**init\_task**并**不是系统通过kernel\_thread的方式**（当然**更不可能是fork**）创建的,而是由内核黑客**静态创建**的.

该进程的描述符在init/init\_task.c中定义，代码片段如下

```c
/* Initial task structure */
struct task_struct init_task = INIT_TASK(init_task);
EXPORT_SYMBOL(init_task);
```

**init\_task描述符**使用**宏INIT\_TASK**对init\_task的**进程描述符进行初始化**，宏INIT\_TASK在[include/linux/init_task.h](http://lxr.free-electrons.com/source/include/linux/init_task.h?v=4.5#L186)文件中

init\_task是Linux内核中的**第一个线程**，它贯穿于整个Linux系统的初始化过程中，该进程也是Linux系统中唯一一个没有用kernel\_thread()函数创建的内核态进程(内核线程)

在**init\_task进程执行后期**，它会**调用kernel\_thread**()函数创建**第一个核心进程kernel\_init**，同时init\_task进程**继续对Linux系统初始化**。在**完成初始化后**，**init\_task**会**退化为cpu\_idle进程**，当**Core 0**的**就绪队列**中**没有其它进程**时，该进程将会**获得CPU运行**。**新创建的1号进程kernel\_init将会逐个启动次CPU**,并**最终创建用户进程**！

备注：**core 0**上的**idle进程**由**init\_task进程退化**而来，而**AP的idle进程**则是**BSP在后面调用fork()函数逐个创建**的

### 2.1.1 进程堆栈init\_thread\_union

init\_task进程使用**init\_thread\_union**数据结构**描述的内存区域**作为**该进程的堆栈空间**，并且**和自身的thread\_info**参数**共用这一内存空间空间**，

>请参见 http://lxr.free-electrons.com/source/include/linux/init_task.h?v=4.5#L193
>
>         .stack          = &init_thread_info,


而**init\_thread\_info**则是一段**体系结构相关的定义**，被定义**在[/arch/对应体系/include/asm/thread_info.h**]中，但是他们大多数为如下定义

```c
#define init_thread_info        (init_thread_union.thread_info)
#define init_stack              (init_thread_union.stack)
```

其中**init\_thread\_union**被定义在init/init\_task.c

```c
/*
 * Initial thread structure. Alignment of this is handled by a special
 * linker map entry.
 */
union thread_union init_thread_union __init_task_data =
        { INIT_THREAD_INFO(init_task) };
```

我们可以发现init\_task是用INIT\_THREAD\_INFO宏进行初始化的,这个才是我们**真正体系结构相关的部分**,他与init\_thread\_info定义在一起，被定义在[/arch/对应体系/include/asm/thread_info.h](http://lxr.free-electrons.com/ident?v=4.5;i=INIT_THREAD_INFO)中，以下为[x86架构的定义](http://lxr.free-electrons.com/source/arch/x86/include/asm/thread_info.h?v=4.5#L65)

> 参见 
> 
> http://lxr.free-electrons.com/source/arch/x86/include/asm/thread_info.h?v=4.5#L65

```c
#define INIT_THREAD_INFO(tsk)                   \
{                                               \
    .task           = &tsk,                 \
    .flags          = 0,                    \
    .cpu            = 0,                    \
    .addr_limit     = KERNEL_DS,            \
}
```

>其他体系结构的定义请参见
>
>[/arch/对应体系/include/asm/thread_info.h](http://lxr.free-electrons.com/ident?v=4.5;i=INIT_THREAD_INFO)中

| 架构 | 定义 |
| ------------- |:-------------:|
| x86 | [arch/x86/include/asm/thread_info.h](http://lxr.free-electrons.com/source/arch/x86/include/asm/thread_info.h?v=4.5#L65) |
| arm64 | [arch/arm64/include/asm/thread_info.h](http://lxr.free-electrons.com/source/arch/arm64/include/asm/thread_info.h?v=4.5#L55) |

init\_thread\_info定义中**的\_\_init\_task\_data**表明该内核栈所在的区域**位于内核映像的init data区**，我们可以通过**编译完内核后**所产生的**System.map**来看到该变量及其对应的逻辑地址

```
cat System.map-3.1.6 | grep init_thread_union
```

![init_thread_union](./images/init_thread_union.png)

### 2.1.2 进程内存空间

**init\_task**的**虚拟地址空间**，也采用同样的方法被定义

由于init\_task是一个**运行在内核空间的内核线程**,因此**其虚地址段mm为NULL**,但是必要时他还是**需要使用虚拟地址**的，因此**avtive\_mm被设置为init\_mm**

> 参见
>
> http://lxr.free-electrons.com/source/include/linux/init_task.h?v=4.5#L202

```c
.mm             = NULL,                                         \
.active_mm      = &init_mm,                                     \
```
其中init\_mm被定义为init\-mm.c中，参见 http://lxr.free-electrons.com/source/mm/init-mm.c?v=4.5#L16

```c
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

# 3 0号进程的演化

## 3.1 rest\_init创建init进程(PID=1)和kthread进程(PID=2)

Linux在**无进程概念的情况下**将一直从初始化部分的代码执行到start\_kernel，然后再到其**最后一个函数调用rest\_init**

大致是在vmlinux的入口**startup\_32(head.S**)中为pid号为0的原始进程**设置了执行环境**，然后**原始进程开始执行start\_kernel**()完成Linux内核的初始化工作。包括初始化页表，初始化中断向量表，初始化系统时间等。

从**rest\_init**开始，Linux开始**产生进程**，因为init\_task是静态制造出来的，pid=0，它试图将**从最早的汇编代码**一直到**start\_kernel的执行**都纳入到**init\_task进程上下文**中。

这个**函数**其实是由**0号进程执行**的, 他就是在这个函数中, 创建了**init进程**和**kthreadd**进程

这部分代码如下：

>参见	
>
>http://lxr.free-electrons.com/source/init/main.c?v=4.5#L386

```c
static noinline void __init_refok rest_init(void)
{
	int pid;

	rcu_scheduler_starting();
	smpboot_thread_init();

    /*
 	* We need to spawn init first so that it obtains pid 1, however
 	* the init task will end up wanting to create kthreads, which, if
 	* we schedule it before we create kthreadd, will OOPS.
 	*/
	kernel_thread(kernel_init, NULL, CLONE_FS);
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);

	/*
 	* The boot idle thread must execute schedule()
 	* at least once to get things moving:
 	*/
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
}
```

1. 调用**kernel\_thread**()创建**1号内核线程**, 该线程**随后转向用户空间**, 演变为**init进程**

2. 调用kernel\_thread()创建**kthreadd内核线程**。

3. init\_idle\_bootup\_task()：**当前0号进程**init\_task最终会**退化成idle进程**，所以这里调用init\_idle\_bootup\_task()函数，让**init\_task进程隶属到idle调度类**中。即选择idle的调度相关函数。

4. **调用schedule**()函数**切换当前进程**，在**调用该函数之前**，Linux系统中**只有两个进程**，即**0号进程init\_task**和**1号进程kernel\_init**，其中kernel\_init进程也是刚刚被创建的。**调用该函数后，1号进程kernel_init将会运行**！

5. 调用cpu\_idle()，0号线程进入idle函数的循环，在该循环中会周期性地检查。

### 3.1.1 创建kernel\_init

在rest\_init函数中，内核将通过下面的代码**产生第一个真正的进程(pid=1**):

```c
kernel_thread(kernel_init, NULL, CLONE_FS);
```

这个进程就是著名的pid为1的init进程，它会继续完成剩下的初始化工作，然后execve(/sbin/init),成为系统中的其他所有进程的祖先。

但是这里我们发现一个问题,**init进程**应该是一个**用户空间的进程**,但是这里却是**通过kernel\_thread的方式创建的**, 哪岂不是式一个永远运行在内核态的内核线程么,它是**怎么演变为真正意义上用户空间的init进程**的？

**1号kernel\_init进程**完成**linux的各项配置(包括启动AP**)后，就会**在/sbin,/etc,/bin寻找init程序**来运行。**该init程序会替换kernel\_init进程**（注意：并**不是创建一个新的进程来运行init程序**，而是一次变身，**使用sys\_execve函数**改变**核心进程的正文段**，将核心进程kernel\_init**转换成用户进程init**），此时处于内核态的1号kernel\_init进程将会转换为用户空间内的1号进程init。**用户进程init**将**根据/etc/inittab**中提供的信息**完成应用程序的初始化调用**。然后**init进程会执行/bin/sh产生shell界面**提供给用户来**与Linux系统进行交互**。
>
>调用init\_post()创建用户模式1号进程。

关于init其他的信息我们这次先不研究，因为我们这篇旨在探究0号进程的详细过程，

### 3.1.2 创建kthreadd

在rest\_init函数中，内核将通过下面的代码产生**第一个kthreadd(pid=2**)

```c
pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

它的任务就是**管理和调度其他内核线程**kernel\_thread,会循环执行一个kthread的函数，该函数的作用就是运行kthread\_create\_list全局链表中维护的kthread,当我们调用kernel\_thread创建的内核线程会被加入到此链表中，因此所有的内核线程都是直接或者间接的以kthreadd为父进程

## 3.2 0号进程演变为idle

```c
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
```

因此我们回过头来看pid=0的进程，在**创建了init进程后**，pid=0的进程**调用cpu\_idle**()演变成了**idle进程**。

0号进程首先执行init\_idle\_bootup\_task,**让init\_task进程隶属到idle调度类中**。即选择idle的调度相关函数。

这个函数被定义在[kernel/sched/core.c](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.5#L5075)中，如下

```c
void init_idle_bootup_task(struct task_struct *idle)
{
	idle->sched_class = &idle_sched_class;
}
```

接着通过schedule\_preempt\_disabled来**执行调用schedule()函数切换当前进程**，在调用该函数之前，Linux系统中只有两个进程，即**0号进程init\_task**和**1号进程kernel\_init**，其中kernel\_init进程也是刚刚被创建的。**调用该函数**后，**1号进程kernel\_init将会运行**

这个函数被定义在[kernel/sched/core.c](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.5#L3342)中，如下

```c
/**
* schedule_preempt_disabled - called with preemption disabled
*
* Returns with preemption disabled. Note: preempt_count must be 1
*/
void __sched schedule_preempt_disabled(void)
{
	sched_preempt_enable_no_resched();
	schedule();
	preempt_disable();
}
```

最后cpu\_startup\_entry**调用cpu\_idle\_loop()，0号线程进入idle函数的循环，在该循环中会周期性地检查**

cpu\_startup\_entry定义在[kernel/sched/idle.c](http://lxr.free-electrons.com/source/kernel/sched/idle.c?v=4.5#L276)

```c
 void cpu_startup_entry(enum cpuhp_state state)
{
	/*
	* This #ifdef needs to die, but it's too late in the cycle to
	* make this generic (arm and sh have never invoked the canary
	* init for the non boot cpus!). Will be fixed in 3.11
	*/
#ifdef CONFIG_X86
    /*
    * If we're the non-boot CPU, nothing set the stack canary up
    * for us. The boot CPU already has it initialized but no harm
    * in doing it again. This is a good place for updating it, as
    * we wont ever return from this function (so the invalid
    * canaries already on the stack wont ever trigger).
    */
    boot_init_stack_canary();
#endif
    arch_cpu_idle_prepare();
    cpu_idle_loop();
}
```

其中cpu\_idle\_loop就是**idle进程的事件循环**，定义在[kernel/sched/idle.c](http://lxr.free-electrons.com/source/kernel/sched/idle.c?v=4.5#L203)

整个过程简单的说就是，**原始进程(pid=0**)创建**init进程(pid=1**),然后演化成**idle进程(pid=0**)。**init进程**为**每个从处理器(运行队列**)创建出一个**idle进程(pid=0**)，然后**演化成/sbin/init**。

# 4 idle的运行与调度

## 4.1 idle的workload--cpu\_idle\_loop

从上面的分析我们知道，**idle**在系统**没有其他就绪的进程可执行**的时候才会**被调度**。不管是**主处理器**，还是**从处理器**，最后都是执行的**cpu\_idle\_loop**()函数

其中cpu\_idle\_loop就是**idle进程的事件循环**，定义在[kernel/sched/idle.c](http://lxr.free-electrons.com/source/kernel/sched/idle.c?v=4.5#L203)，**早期的版本**中提供的是[cpu\_idle](http://lxr.free-electrons.com/ident?v=3.9;i=cpu_idle)，但是**这个函数是完全依赖于体系结构**的，不利用架构的分层，因此在新的内核中更新为更加通用的cpu\_idle\_loop，由他来**调用体系结构相关的代码**

所以我们来看看cpu\_idle\_loop做了什么事情。

因为**idle进程**中并不执行什么有意义的任务，所以通常考虑的是两点

1. **节能**

2. **低退出延迟**。

其代码如下

```c
/*
 * Generic idle loop implementation
 *
 * Called with polling cleared.
 */
static void cpu_idle_loop(void)
{
        while (1) {
                /*
                 * If the arch has a polling bit, we maintain an invariant:
                 *
                 * Our polling bit is clear if we're not scheduled (i.e. if
                 * rq->curr != rq->idle).  This means that, if rq->idle has
                 * the polling bit set, then setting need_resched is
                 * guaranteed to cause the cpu to reschedule.
                 */

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

                        /*
                         * In poll mode we reenable interrupts and spin.
                         *
                         * Also if we detected in the wakeup from idle
                         * path that the tick broadcast device expired
                         * for us, we don't want to go deep idle as we
                         * know that the IPI is going to arrive right
                         * away
                         */
                        if (cpu_idle_force_poll || tick_check_broadcast_expired())
                                cpu_idle_poll();
                        else
                                cpuidle_idle_call();

                        arch_cpu_idle_exit();
                }

                /*
                 * Since we fell out of the loop above, we know
                 * TIF_NEED_RESCHED must be set, propagate it into
                 * PREEMPT_NEED_RESCHED.
                 *
                 * This is required because for polling idle loops we will
                 * not have had an IPI to fold the state for us.
                 */
                preempt_set_need_resched();
                tick_nohz_idle_exit();
                __current_clr_polling();

                /*
                 * We promise to call sched_ttwu_pending and reschedule
                 * if need_resched is set while polling is set.  That
                 * means that clearing polling needs to be visible
                 * before doing these things.
                 */
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

3. Idle进程为最低优先级，且不参与调度，只是在运行队列为空的时候才被调度。

4. Idle**循环等待need\_resched置位**。**默认使用hlt节能**。

希望通过本文你能全面了解linux内核中idle知识。

## 4.2 idle的调度和运行时机

我们知道, linux进程的**调度顺序**是按照**rt实时进程(rt调度器),normal普通进程(cfs调度器)，和idle**的顺序来调度的

那么可以试想**如果rt和cfs都没有可以运行**的任务，那么**idle才可以被调度**，那么他是通过**怎样的方式实现**的呢？

由于我们还没有讲解调度器的知识, 所有我们只是简单讲解一下

在**normal的调度类,cfs公平调度器**sched\_fair.c中, 我们可以看到

```c
static const struct sched_class fair_sched_class = {
    .next = &idle_sched_class,
```

也就是说，如果**系统中没有普通进程**，那么会**选择下个调度类优先级的进程**，即**使用idle\_sched\_class调度类进行调度的进程**

当**系统空闲**的时候，最后就是**调用idle的pick\_next\_task函数**，被定义在/kernel/sched/idle\_task.c中

>参见
>
>http://lxr.free-electrons.com/source/kernel/sched/idle_task.c?v=4.5#L27

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

# 5 idle进程总结

系统允许一个进程创建新进程，新进程即为子进程，子进程还可以创建新的子进程，形成进程树结构模型。整个linux系统的所有进程也是一个树形结构。**树根是系统自动构造的(或者说是由内核黑客手动创建的)**，即**在内核态下执行的0号进程**，它是**所有进程的远古先祖**。

在smp系统中，每个处理器单元有独立的一个运行队列，而每个运行队列上又有一个idle进程，即**有多少处理器单元，就有多少idle进程**。

1. idle进程其pid=0，其前身是系统创建的第一个进程(我们称之为init\_task)，也是**唯一一个没有通过fork或者kernel\_thread产生的进程**。

2. **init\_task**是内核中所有进程、线程的task\_struct雏形，它是在内核初始化过程中，通过静态定义构造出了一个task\_struct接口，取名为init\_task，然后在内核初始化的后期，在rest\_init()函数中通过kernel\_thread创建了两个内核线程**内核init线程，kthreadd内核线程**,前者后来通过演变，进入用户空间，成为所有用户进程的先祖,而后者则成为所有内核态其他守护线程的父线程, 负责接手内核线程的创建工作

3. 然后init\_task通过变更调度类为sched_idle等操作演变成为**idle进程**, 此时系统中只有0(idle), 1(init), 2(kthreadd)3个进程, 然后执行一次进程调度, 必然**切换当前进程到到init**

# 6 附录\-\-rest\_init的执行解析

| rest_init 流程 | 说明 |
| ------------- |:-------------|
| rcu\_scheduler\_starting	| 启动Read\-Copy Update,会调用num\_online\_cpus确认目前只有bootstrap处理器在运作,以及调用nr\_context\_switches确认在启动RCU前,没有进行过Contex\-Switch,最后就是设定rcu\_scheduler\_active=1启动RCU机制. RCU在多核心架构下,不同的行程要读取同一笔资料内容/结构,可以提供高效率的同步与正确性. 在这之后就可以使用 rcu\_read\_lock/rcu\_read\_unlock了 |
| 产生Kernel Thread kernel\_init | Kernel Thread函式 kernel\_init实例在init/main.c中, init Task PID=1,是内核第一个产生的Task. 产生后,会阻塞在wait\_for\_completion处,等待kthreadd\_done Signal,以便往后继续执行下去. |
|产生Kernel Thread kthreadd | Kernel Thread函式 kthreadd实例在kernel/kthread.c中, kthreadd Task PID=2,是内核第二个产生的Task. |
| find\_task\_by\_pid\_ns	| 实例在kernel/pid.c中,调用函数find\_task\_by\_pid\_ns,并传入参数kthreadd的PID 2与PID NameSpace (struct pid\_namespace init\_pid\_ns)取回PID 2的Task Struct. |
| complete	| 实例在kernel/sched.c中, 会发送kthreadd\_done Signal,让 kernel\_init(也就是 init task)可以往后继续执行. |
| init\_idle\_bootup\_task	| 实例在kernel/sched.c中, 设定目前启动的Task为IDLE Task.(idle\->sched\_class = &idle\_sched\_class), 而struct sched\_class idle\_sched\_class的定义在kernel/sched\_idletask.c中.在**Linux下IDLETask并不占用PID(也可以把它当作是PID 0**),每个处理器都会有这样的IDLE Task,用来在没有行程排成时,让处理器掉入执行的.而最基础的省电机制,也可透过IDLE Task来进行. (包括让系统可以关闭必要的周边电源与Clock Gating). |
| schedule\_preempt\_disabled(); | 启动一次Linux Kernel Process的排成Context\-Switch调度机制, 从而使得kernel\_init即1号进程获得处理机 |
| cpu\_startup\_entry | 完成工作后, 调用cpu\_idle\_loop()使得idle进程进入自己的事件处理循环 |