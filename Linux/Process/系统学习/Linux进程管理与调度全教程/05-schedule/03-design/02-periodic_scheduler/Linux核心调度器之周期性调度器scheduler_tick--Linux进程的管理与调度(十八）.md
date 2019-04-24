
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 两个调度器](#1-两个调度器)
* [2 周期性调度器](#2-周期性调度器)
	* [2.1 周期性调度器主流程](#21-周期性调度器主流程)
	* [2.2 更新统计量](#22-更新统计量)
	* [2.3 激活进程所属调度类的周期性调度器](#23-激活进程所属调度类的周期性调度器)
* [3 周期性调度器的激活](#3-周期性调度器的激活)
	* [3.1 定时器周期性的激活调度器](#31-定时器周期性的激活调度器)
	* [3.2 早期实现](#32-早期实现)
	* [3.3 目前实现](#33-目前实现)
* [4 参考](#4-参考)

<!-- /code_chunk_output -->

# 1 两个调度器

我们前面提到linux有两种方法激活调度器

- 一种是直接的, 比如进程打算睡眠或出于其他原因放弃CPU

- 另一种是通过周期性的机制, 以固定的频率运行, 不时的检测是否有必要

因而内核提供了两个调度器**主调度器**，**周期性调度器**，分别实现如上工作,两者合在一起就组成了**核心调度器(core scheduler)**,也叫**通用调度器(generic scheduler)**.

他们都**根据进程的优先级分配CPU时间**,因此这个过程就叫做**优先调度**,我们将在本节主要讲解**核心调度器的设计和优先调度的实现方式**.

而我们的**周期性调度器**以**固定的频率激活**负责**当前进程调度类的周期性调度方法**,以**保证系统的并发性**

# 2 周期性调度器

周期性调度器在**scheduler\_tick**()中实现.如果系统正在活动中,内核会**按照频率HZ自动调用该函数**.如果**没有进程在等待调度**, 那么在计算机电力供应不足的情况下,内核**将关闭该调度器以减少能耗(！！！**).这对于我们的嵌入式设备或者手机终端设备的电源管理是很重要的.

## 2.1 周期性调度器主流程

scheduler\_tick函数定义在[kernel/sched/core.c, L2910](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.6#L2910)中, 它有两个主要任务

1. **更新相关统计量**

管理内核中的与整个系统和各个进程的调度相关的统计量. 其间执行的**主要操作是对各种计数器+1**

2. 激活负责**当前进程调度类的周期性调度方法**

**检查进程执行的时间**是否超过了它**对应的ideal\_runtime**，如果超过了，则告诉系统，需要**启动主调度器(schedule)进行进程切换**。(注意thread\_info:preempt\_count、thread\_info:flags (TIF\_NEED\_RESCHED))

```c
[kernel/sched/core.c]
void scheduler_tick(void)
{
    /*  1.  获取当前cpu上的全局就绪队列rq和当前运行的进程curr  */

    /*  1.1 在于SMP的情况下，获得当前CPU的ID。如果不是SMP，那么就返回0  */
    int cpu = smp_processor_id();

    /*  1.2 获取cpu的全局就绪队列rq, 每个CPU都有一个就绪队列rq  */
    struct rq *rq = cpu_rq(cpu);

    /*  1.3 获取就绪队列上正在运行的进程curr  */
    struct task_struct *curr = rq->curr;

    sched_clock_tick();

	/*  2 更新rq上的统计信息, 并执行进程对应调度类的周期性的调度  */

    /*  加锁 */
    raw_spin_lock(&rq->lock);

    /*  2.1 更新rq的当前时间戳.即使rq->clock变为当前时间戳  */
    update_rq_clock(rq);

    /*  2.2 执行当前运行进程所在调度类的task_tick函数进行周期性调度  */
    curr->sched_class->task_tick(rq, curr, 0);

    /*  2.3 更新rq的负载信息,  即就绪队列的cpu_load[]数据
     *  本质是讲数组中先前存储的负荷值向后移动一个位置，
     *  将当前负荷记入数组的第一个位置 */
    update_cpu_load_active(rq);

    /*  2.4 更新cpu的active count活动计数
     *  主要是更新全局cpu就绪队列的calc_load_update*/
    calc_global_load_tick(rq);

    /* 解锁 */
    raw_spin_unlock(&rq->lock);

    /* 与perf计数事件相关 */
    perf_event_task_tick();

#ifdef CONFIG_SMP

     /* 当前CPU是否空闲 */
    rq->idle_balance = idle_cpu(cpu);

    /* 如果到是时候进行周期性负载平衡则触发SCHED_SOFTIRQ */
    trigger_load_balance(rq);

#endif

    rq_last_tick_reset(rq);
}
```

## 2.2 更新统计量

| 函数 | 描述 | 定义 |
| ------------- |:-------------:|:-------------:|
| update\_rq\_clock | 处理**就绪队列时钟**的更新, 本质上就是**增加struct rq当前实例的时钟时间戳** | [sched/core.c, L98](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.6#L98)
| update\_cpu\_load\_active | 负责**更新就绪队列的cpu\_load数组**,其本质上相当于**将数组中先前存储的负荷值向后移动一个位置**,将**当前就绪队列的负荷记入数组的第一个位置**. 另外该函数还引入一些取平均值的技巧, 以确保符合数组的内容不会呈现太多的不联系跳读. | [kernel/sched/fair.c, L4641](http://lxr.free-electrons.com/source/kernel/sched/fair.c?v=4.6#L4641) |
| calc\_global\_load\_tick | **更新cpu的活动计数**, 主要是更新**全局cpu就绪队列的calc\_load\_update** |  [kernel/sched/loadavg.c, L382](http://lxr.free-electrons.com/source/kernel/sched/loadavg.c?v=4.6#L378) |

## 2.3 激活进程所属调度类的周期性调度器

由于**调度器的模块化结构,主体工程其实很简单**,在**更新统计信息的同时**,内核**将真正的调度工作委托给了特定的调度类方法**

内核先找到了**就绪队列上当前运行的进程curr**,然后**调用curr所属调度类sched\_class的周期性调度方法task\_tick**

即

```c
curr->sched_class->task_tick(rq, curr, 0);
```

task\_tick的实现方法**取决于底层的调度器类**,例如**完全公平调度器**会在该方法中检测**是否进程已经运行了太长的时间**, 以避免过长的延迟,注意此处的做法与之前就的**基于时间片的调度方法**有本质区别,旧的方法我们称之为到期的时间片, 而**完全公平调度器CFS中则不存在所谓的时间片概念**.

目前我们的内核中的3个调度器类`struct sched_entity`,  `struct sched_rt_entity`, 和`struct sched_dl_entity` dl, 我们针对当前内核中实现的调度器类分别列出其周期性调度函数task\_tick

| 调度器类 | task\_tick操作 | task\_tick函数定义 |
| ------- |:-------|:-------|
| stop\_sched\_class | | [kernel/sched/stop\_task.c, line 77, task\_tick\_stop](http://lxr.free-electrons.com/source/kernel/sched/stop_task.c?v=4.6#L77) |
| dl\_sched\_class | | [kernel/sched/deadline.c, line 1192, task\_tick\_dl](http://lxr.free-electrons.com/source/kernel/sched/deadline.c?v=4.6#L1192)|
| rt\_sched\_class | | [/kernel/sched/rt.c, line 2227, task\_tick\_rt](http://lxr.free-electrons.com/source/kernel/sched/rt.c?v=4.6#L2227)|
| fail\_sched\_class | | [kernel/sched/fair.c, line 8116, task\_tick\_fail](http://lxr.free-electrons.com/source/kernel/sched/fair.c?v=4.6#L8116) |
| idle\_sched\_class | | [kernel/sched/idle_task.c, line 53, task\_tick\_idle](http://lxr.free-electrons.com/source/kernel/sched/idle_task.c?v=4.6#L53) |

- 如果当前进程是**完全公平队列**中的进程,则首先根据**当前就绪队列中的进程数**算出一个**延迟时间间隔**，大概**每个进程分配2ms时间**，然后按照该进程在队列中的**总权重中占的比例**，算出它**该执行的时间X**，如果**该进程执行物理时间超过了X**，则**激发延迟调度**；如果没有超过X，但是**红黑树就绪队列中下一个进程优先级更高**，即curr->vruntime-leftmost->vruntime > X,也**将延迟调度**

>**延迟调度**的**真正调度过程**在：**schedule中实现**，会**按照调度类顺序和优先级挑选出一个最高优先级的进程执行**

- 如果当前进程是**实时调度类**中的进程：则如果**该进程是SCHED\_RR**，则**递减时间片为[HZ/10**]，**到期，插入到队列尾部**，并**激发延迟调度**，如果是**SCHED\_FIFO**，则什么也不做，**直到该进程执行完成**

如果**当前进程希望被重新调度**,那么**调度类方法**会在**task\_struct**中设置**TIF\_NEED\_RESCHED标志**,以表示该请求, 而内核将会在**接下来的适当实际完成此请求**.

# 3 周期性调度器的激活

## 3.1 定时器周期性的激活调度器

定时器是Linux提供的一种**定时服务的机制**. 它在**某个特定的时间唤醒某个进程**，来做一些工作.

在**低分辨率定时器**的**每次时钟中断**完成**全局统计量更新**后,**每个cpu(！！！**)在**软中断中**执行以下操作

- 更新**该cpu上当前进程内核态**、**用户态**使用时间**xtime\_update**
- 调用**该cpu上的定时器函数**
- 启动**周期性定时器**（**scheduler\_tick**）完成**该cpu**上任务的**周期性调度工作**；

在**支持动态定时器(！！！**)的系统中，**可以关闭该调度器(没有进程在等待调度时候！！！**)，从而**进入深度睡眠过程**；scheduler\_tick查看**当前进程是否运行太长时间**，如果是，将**进程的TIF\_NEED\_RESCHED置位**，然后**在中断返回**时，调用**schedule**，进行**进程切换**操作

```c
//  http://lxr.free-electrons.com/source/arch/arm/kernel/time.c?v=4.6#L74
/*
* Kernel system timer support.
*/
void timer_tick(void)
{
    profile_tick(CPU_PROFILING);
    xtime_update(1);
#ifndef CONFIG_SMP
    update_process_times(user_mode(get_irq_regs()));
#endif
}

//  http://lxr.free-electrons.com/source/kernel/time/timer.c?v=4.6#L1409
/*
 * Called from the timer interrupt handler to charge one tick to the current
 * process.  user_tick is 1 if the tick is user time, 0 for system.
 */
void update_process_times(int user_tick)
{
    struct task_struct *p = current;

    /* Note: this timer irq context must be accounted for as well. */
    account_process_tick(p, user_tick);
    run_local_timers();
    rcu_check_callbacks(user_tick);
#ifdef CONFIG_IRQ_WORK
    if (in_irq())
        irq_work_tick();
#endif
    // 周期性调度器
    scheduler_tick();
    run_posix_cpu_timers(p);
}
```

## 3.2 早期实现

**Linux初始化**时, **init\_IRQ**()函数设定**8253的定时周期为10ms(一个tick值**).同样，在**初始化**时,**time\_init**()用**setup\_irq**()设置**时间中断向量irq0**, **中断服务程序为timer\_interrupt**.

在2.4版内核及较早的版本当中,**定时器的中断处理采用底半机制**,**底半处理函数的注册**在**start\_kernel**()函数中调用**sechd\_init**(), 在这个函数中又调用init\_bh(TIMER\_BH, timer\_bh)注册了定时器的底半处理函数.然后系统才**调用time\_init()来注册定时器的中断向量和中断处理函数**.

在中断处理函数timer\_interrupt()中，主要是调用do\_timer()函数完成工作。do\_timer()函数的主要功能就是调用**mark\_bh()产生软中断**，随后处理器会在合适的时候调用定时器底半处理函数timer\_bh()。在timer\_bh()中, 实现了**更新定时器**的功能. 2.4.23版的do\_timer()函数代码如下（经过简略）：

```c
void do_timer(struct pt_regs *regs)
{
       (*(unsigned long *)&jiffies)++;
       update_process_times(user_mode(regs));
       mark_bh(TIMER_BH);
}
```

## 3.3 目前实现

而在**内核2.6版本**以后，**定时器中断处理**采用了**软中断机制**而**不是底半机制**。**时钟中断处理函数**仍然为**timer\_interrupt**()\-> do\_timer\_interrupt()\-> do\_timer\_interrupt\_hook()\-> do\_timer()。不过do\_timer()函数的实现有所不同
```c
void do_timer(struct pt_regs *regs)
{
       jiffies_64++;
       update_process_times(user_mode(regs));
       update_times();
}
```

>更详细的实现linux-2.6
>
[Linux中断处理之时钟中断（一）](http://www.bianceng.cn/OS/Linux/201111/31272_4.htm)
>
>[（原创）linux内核进程调度以及定时器实现机制](http://blog.csdn.net/joshua_yu/article/details/591038)

# 4 参考

[进程管理与调度5 -- 进程调度、进程切换原理详解](http://wanderer-zjhit.blogbus.com/logs/156738683.html)

[Linux中断处理之时钟中断（一）](http://www.bianceng.cn/OS/Linux/201111/31272_4.htm)

[（原创）linux内核进程调度以及定时器实现机制](http://blog.csdn.net/joshua_yu/article/details/591038)