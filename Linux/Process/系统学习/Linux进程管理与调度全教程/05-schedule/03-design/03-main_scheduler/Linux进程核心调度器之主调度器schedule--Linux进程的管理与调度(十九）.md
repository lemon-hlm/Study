[TOC]

- 1 前景回顾
    - 1.1 进程调度
    - 1.2 进程的分类
    - 1.3 linux调度器的演变
    - 1.4 Linux的调度器组成
        - 1.4.1 2个调度器
        - 1.4.2 6种调度策略
        - 1.4.3 5个调度器类
        - 1.4.4 3个调度实体
- 2 主调度器
    - 2.1 调度函数的\_\_sched前缀
    - 2.2 schedule函数
        - 2.2.1 schedule主框架
        - 2.2.2 sched\_submit\_work避免死锁
        - 2.2.3 preempt\_disable和sched\_preempt\_enable\_no\_resched开关内核抢占
    - 2.3 \_\_schedule开始进程调度
        - 2.3.1 \_\_schedule函数主框架
        - 2.3.2 pick\_next\_task选择抢占的进程
    - 2.4 context\_switch进程上下文切换
        - 2.4.1 进程上下文切换
        - 2.4.2 context\_switch流程
        - 2.4.3 switch\_mm切换进程虚拟地址空间
        - 2.4.4 switch\_to切换进程堆栈和寄存器
    - 2.5 need\_resched, TIF\_NEED\_RESCHED标识与用户抢占
        - 2.5.1 need\_resched标识TIF\_NEED\_RESCHED
        - 2.5.2 用户抢占和内核抢占
- 3 总结
    - 3.1 schedule调度流程
    - 3.2 \_\_schedule如何完成内核抢占
    - 3.3 调度的内核抢占和用户抢占

# 1 前景回顾

## 1.1 进程调度

内存中保存了对每个进程的唯一描述, 并通过若干结构与其他进程连接起来.

调度器面对的情形就是这样, 其任务是在程序之间共享CPU时间, 创造并行执行的错觉, 该任务分为两个不同的部分, 其中一个涉及调度策略, 另外一个涉及上下文切换.

## 1.2 进程的分类

linux把进程区分为**实时进程**和**非实时进程**, 其中非实时进程进一步划分为交互式进程和批处理进程

根据进程的不同分类Linux采用不同的调度策略.

对于实时进程，采用FIFO, Round Robin或者Earliest Deadline First (EDF)最早截止期限优先调度算法|的调度策略.

## 1.3 linux调度器的演变

| 字段 | 版本 |
| ------------- |:-------------:|
| O(n)的始调度算法 | linux-0.11~2.4 |
| O(1)调度器 | linux-2.5 |
| CFS调度器 | linux-2.6~至今 |

## 1.4 Linux的调度器组成

### 1.4.1 2个调度器

可以用**两种方法来激活调度**

- 一种是**直接的**, 比如**进程打算睡眠**或出于**其他原因放弃CPU**

- 另一种是通过**周期性的机制**, 以**固定的频率运行**, 不时的检测是否有必要

因此当前**linux的调度程序**由**两个调度器组成**：

- **主调度器**

- **周期性调度器**

两者又统称为**通用调度器(generic scheduler)**或**核心调度器(core scheduler)**

并且**每个调度器**包括两个内容：**调度框架**(其实质就是**两个函数框架**)及**调度器类**

他们都**根据进程的优先级分配CPU时间**,因此这个过程就叫做**优先调度**,我们将在本节主要讲解**主调度的设计和实现方式**

在内核中的许多地方, 如果要**将CPU分配**给与当前活动进程不同的**另一个进程**,都会**直接调用主调度器函数schedule**;从**系统调用返回**后,内核也会**检查当前进程是否设置了重调度标志TLF\_NEDD\_RESCHED**

### 1.4.2 6种调度策略

linux内核目前实现了**6种调度策略(即调度算法**), 用于对不同类型的进程进行调度,或者支持某些特殊的功能

- **SCHED\_NORMAL**和**SCHED\_BATCH**调度**普通的非实时进程**

- **SCHED\_FIFO**和**SCHED\_RR**和**SCHED\_DEADLINE**则采用不同的调度策略**调度实时进程**

- **SCHED\_IDLE**则在**系统空闲时调用idle进程**.

### 1.4.3 5个调度器类

而依据其调度策略的不同实现了**5个调度器类**,一个调度器类可以用**一种或者多种调度策略**调度某一类进程, 也可以用于特殊情况或者调度特殊功能的进程.

其所属进程的优先级顺序为
```c
stop_sched_class -> dl_sched_class -> rt_sched_class -> fair_sched_class -> idle_sched_class
```

### 1.4.4 3个调度实体

调度器**不限于调度进程**, 还可以调度更大的实体, 比如实现**组调度**.

这种一般性要求**调度器不直接操作进程**,而是**处理可调度实体**,因此需要一个通用的数据结构描述这个调度实体,即**seched\_entity结构**,其实际上就代表了一个**调度对象**，可以为**一个进程**，也可以为**一个进程组**.

linux中针对当前**可调度的实时**和**非实时进程**, 定义了类型为**seched\_entity的3个调度实体**

- **sched\_dl\_entity** 采用**EDF算法调度的实时调度实体**

- **sched\_rt\_entity** 采用**Roound-Robin或者FIFO算法调度的实时调度实体**

- **sched\_entity** 采用CFS算法调度的**普通非实时进程的调度实体**

# 2 主调度器

在内核中的许多地方, 如果要将CPU分配给与当前活动进程不同的另一个进程,都会**直接调用主调度器函数schedule**(); 

从**系统调用返回**后, 内核也会检查当前进程是否设置了**重调度标志TLF\_NEDD\_RESCHED**

例如, 前述的**周期性调度器的scheduler\_tick()就会设置该标志**,如果是这样则**内核会调用schedule**,该函数假定当前活动进程一定会被另一个进程取代.

## 2.1 调度函数的\_\_sched前缀

在详细论述schedule之前, 需要**说明一下\_\_sched前缀**,该前缀用于**可能调用schedule的函数**,包括**schedule本身**.

\_\_sched前缀的声明, 在[include/linux/sched.h, L416](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.6#L416), 如下所示

``` c
/* Attach to any functions which should be ignored in wchan output. */
#define __sched         __attribute__((__section__(".sched.text")))
```

\_\_attribute\_\_((\_\_section\_("...")))是一个**gcc的编译属性**,其目的在于将**相关的函数的代码编译之后**, 放到**目标文件的特定的段内**,**即.sched.text**中.该信息使得内核在**显示栈转储活类似信息**时, 忽略所有与调度相关的调用. 由于调度函数调用不是普通代码流程的一部分, 因此在这种情况下是没有意义的.

用它修饰函数的方式如下

```c
void __sched some_function(args, ...)
{
	......
	schedule();
    ......
}
```

## 2.2 schedule函数

### 2.2.1 schedule主框架

schedule()就是**主调度器的函数**, 在内核中的许多地方,如果要将CPU分配给与当前活动进程不同的另一个进程, 都会直接调用主调度器函数schedule().

该函数完成**如下工作**

1. 确定**当前就绪队列**, 并在保存一个指向**当前(仍然)活动进程的task\_struct指针**

2. **检查死锁**, **关闭内核抢占**后**调用\_\_schedule完成内核调度**

3. **恢复内核抢占**, 然后检查**当前进程是否设置了重调度标志TLF\_NEDD\_RESCHED**, 如果该进程被其他进程设置了**TIF\_NEED\_RESCHED标志**, 则**函数重新执行进行调度**

该函数定义在[kernel/sched/core.c], 如下所示

```c
asmlinkage __visible void __sched schedule(void)
{

	/*  获取当前的进程  */
    struct task_struct *tsk = current;

    /*  避免死锁 */
    sched_submit_work(tsk);
    do {
        /* 关闭内核抢占 */
        preempt_disable();									
        /* 完成调度 */
        __schedule(false);									
        /* 开启内核抢占 */
        sched_preempt_enable_no_resched();	 
    /* 如果该进程被其他进程设置了TIF_NEED_RESCHED标志，则函数重新执行进行调度 */
    } while (need_resched());
}
EXPORT_SYMBOL(schedule);
```

### 2.2.2 sched\_submit\_work()避免死锁

该函数定义在[kernel/sched/core.c], 如下所示

```c
static inline void sched_submit_work(struct task_struct *tsk)
{
	/* 检测tsk->state是否为0 （runnable）, 若为运行态时则返回，
	 * tsk_is_pi_blocked(tsk),检测tsk的死锁检测器是否为空，若非空的话就return */
    if (!tsk->state || tsk_is_pi_blocked(tsk))
        return;
    /* 然后检测是否需要刷新plug队列，用来避免死锁 */
    if (blk_needs_flush_plug(tsk))
        blk_schedule_flush_plug(tsk);
}
```

### 2.2.3 preempt\_disable和sched\_preempt\_enable\_no\_resched开关内核抢占

**内核抢占**

Linux除了**内核态**外还有**用户态**。**用户程序的上下文**属于**用户态**，**系统调用和中断处理例程上下文属于内核态(！！！**).如果**一个进程**在**用户态**时**被其他进程抢占了CPU则成发生了用户态抢占**, 而如果**此时进程进入了内核态**,则内核此时代替进程执行,如果**此时发了抢占**,我们就说发生了**内核抢占**.

内核抢占是Linux 2.6以后引入的一个重要的概念

我们说：如果**进程正执行内核函数**时，即它**在内核态运行**时，允许发生**内核切换(被替换的进程是正执行内核函数的进程**)，这个内核就是**抢占**的。

抢占内核的主要特点是：一个在**内核态运行的进程**，当且仅当在**执行内核函数期间被另外一个进程取代**。

这与用户态的抢占有本质区别.

内核为了支撑内核抢占,提供了很多机制和结构,必要时候**开关内核抢占**也是必须的,这些函数定义在[include/linux/preempt.h, L145](http://lxr.free-electrons.com/source/include/linux/preempt.h?v=4.6#L145)

```c
#define preempt_disable() \
do { \
    preempt_count_inc(); \
    barrier(); \
} while (0)

#define sched_preempt_enable_no_resched() \
do { \
    barrier(); \
    preempt_count_dec(); \
} while (0)
```

## 2.3 \_\_schedule开始进程调度

\_\_schedule完成了真正的调度工作, 其定义在[kernel/sched/core.c, L3103](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.6#L3103), 如下所示

### 2.3.1 \_\_schedule函数主框架

```c
static void __sched notrace __schedule(bool preempt)
{
    struct task_struct *prev, *next;
    unsigned long *switch_count;
    struct rq *rq;
    int cpu;

    /*  ==1==  
        找到当前cpu上的就绪队列rq
        并将正在运行的进程curr保存到prev中  */
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    prev = rq->curr;

    if (unlikely(prev->state == TASK_DEAD))
        preempt_enable_no_resched_notrace();
    
    /*  如果禁止内核抢占，而又调用了cond_resched就会出错
     *  这里就是用来捕获该错误的  */
    schedule_debug(prev);

    if (sched_feat(HRTICK))
        hrtick_clear(rq);

    /*  关闭本地中断  */
    local_irq_disable();

    /*  更新全局状态，
     *  标识当前CPU发生上下文的切换  */
    rcu_note_context_switch();

    smp_mb__before_spinlock();
    /*  锁住该队列  */
    raw_spin_lock(&rq->lock);
    lockdep_pin_lock(&rq->lock);

    rq->clock_skip_update <<= 1; /* promote REQ to ACT */

    /*  切换次数记录, 默认认为非主动调度计数(抢占)  */
    switch_count = &prev->nivcsw;
    
    /*
     *  scheduler检查prev的状态state和内核抢占表示
     *  如果prev是不可运行的, 并且在内核态没有被抢占
     *  
     *  此时当前进程不是处于运行态, 并且不是被抢占
     *  此时不能只检查抢占计数
     *  因为可能某个进程(如网卡轮询)直接调用了schedule
     *  如果不判断prev->stat就可能误认为task进程为RUNNING状态
     *  到达这里，有两种可能，一种是主动schedule, 另外一种是被抢占
     *  被抢占有两种情况, 一种是时间片到点, 一种是时间片没到点
     *  时间片到点后, 主要是置当前进程的need_resched标志
     *  接下来在时钟中断结束后, 会preempt_schedule_irq抢占调度
     *  
     *  那么我们正常应该做的是应该将进程prev从就绪队列rq中删除, 
     *  但是如果当前进程prev有非阻塞等待信号, 
     *  并且它的状态是TASK_INTERRUPTIBLE
     *  我们就不应该从就绪队列总删除它 
     *  而是配置其状态为TASK_RUNNING, 并且把他留在rq中

    /*  如果内核态没有被抢占, 并且内核抢占有效
        即是否同时满足以下条件：
        1  该进程处于停止状态
        2  该进程没有在内核态被抢占 */
    if (!preempt && prev->state)
    {

        /*  如果当前进程有非阻塞等待信号，并且它的状态是TASK_INTERRUPTIBLE  */
        if (unlikely(signal_pending_state(prev->state, prev)))
        {
            /*  将当前进程的状态设为：TASK_RUNNING  */
            prev->state = TASK_RUNNING;
        }
        else   /*  否则需要将prev进程从就绪队列中删除*/
        {
            /*  将当前进程从runqueue(运行队列)中删除  */
            deactivate_task(rq, prev, DEQUEUE_SLEEP);

            /*  标识当前进程不在runqueue中  */
            prev->on_rq = 0;

            /*
             * If a worker went to sleep, notify and ask workqueue
             * whether it wants to wake up a task to maintain
             * concurrency.
             */
            if (prev->flags & PF_WQ_WORKER) {
                struct task_struct *to_wakeup;

                to_wakeup = wq_worker_sleeping(prev);
                if (to_wakeup)
                    try_to_wake_up_local(to_wakeup);
            }
        }
        /*  如果不是被抢占的，就累加主动切换次数  */
        switch_count = &prev->nvcsw;
    }

    /*  如果prev进程仍然在就绪队列上没有被删除  */
    if (task_on_rq_queued(prev))
        update_rq_clock(rq);  /*  跟新就绪队列的时钟  */

    /*  挑选一个优先级最高的任务将其排进队列  */
    next = pick_next_task(rq, prev);
    /*  清除pre的TIF_NEED_RESCHED标志  */
    clear_tsk_need_resched(prev);
    /*  清楚内核抢占标识  */
    clear_preempt_need_resched();

    rq->clock_skip_update = 0;

    /*  如果prev和next非同一个进程  */
    if (likely(prev != next))
    {
        rq->nr_switches++;  /*  队列切换次数更新  */
        rq->curr = next;    /*  将next标记为队列的curr进程  */
        ++*switch_count;    /* 进程切换次数更新  */

        trace_sched_switch(preempt, prev, next);
        /*  进程之间上下文切换    */
        rq = context_switch(rq, prev, next); /* unlocks the rq */
    }
    else    /*  如果prev和next为同一进程，则不进行进程切换  */
    {
        lockdep_unpin_lock(&rq->lock);
        raw_spin_unlock_irq(&rq->lock);
    }

    balance_callback(rq);
}
STACK_FRAME_NON_STANDARD(__schedule); /* switch_to() */
```

### 2.3.2 pick\_next\_task选择抢占的进程

内核从**cpu的就绪队列**中选择一个**最合适的进程来抢占CPU**

```c
next = pick_next_task(rq);
```

**全局的pick\_next\_task函数**会从按照**优先级遍历所有调度器类**的**pick\_next\_task函数**,去查找**最优的那个进程**, 当然因为**大多数情况**下,系统中全是**CFS调度的非实时进程**,因而linux内核也有一些优化的策略

其执行流程如下

- 如果**当前cpu上所有的进程**都是**cfs调度的普通非实时进程**,则直接用cfs调度,如果**无程序可调度则调度idle进程**

- 否则从**优先级最高的调度器类sched\_class\_highest**(目前是stop\_sched\_class)开始**依次遍历所有调度器类的pick\_next\_task函数**, 选择最优的那个进程执行

其定义在[kernel/sched/core.c, line 3068](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.6#L3064), 如下所示

```c
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev)
{
    const struct sched_class *class = &fair_sched_class;
    struct task_struct *p;

    /*
     * 如果待被调度的进程prev是隶属于CFS的普通非实时进程
     * 而当前cpu的全局就绪队列rq中的进程数与cfs_rq的进程数相等
     * 则说明当前cpu上的所有进程都是由cfs调度的普通非实时进程
     *
     * 那么我们选择最优进程的时候
     * 就只需要调用cfs调度器类fair_sched_class的选择函数pick_next_task
     * 就可以找到最优的那个进程p
     */
    /*  如果当前所有的进程都被cfs调度, 没有实时进程  */
    if (likely(prev->sched_class == class &&
           rq->nr_running == rq->cfs.h_nr_running))
    {
        /*  调用cfs的选择函数pick_next_task找到最优的那个进程p*/
        p = fair_sched_class.pick_next_task(rq, prev);
        /*  #define RETRY_TASK ((void *)-1UL)有被其他调度气找到合适的进程  */
        if (unlikely(p == RETRY_TASK))
            goto again; /*  则遍历所有的调度器类找到最优的进程 */

        /* assumes fair_sched_class->next == idle_sched_class */
        if (unlikely(!p))   /*  如果没有进程可被调度  */
            p = idle_sched_class.pick_next_task(rq, prev); /*  则调度idle进程  */

        return p;
    }

/*  进程中所有的调度器类, 是通过next域链接域链接在一起的
 *  调度的顺序为stop -> dl -> rt -> fair -> idle 
 *  again出的循环代码会遍历他们找到一个最优的进程  */
again:
    for_each_class(class)
    {
        p = class->pick_next_task(rq, prev);
        if (p)
        {
            if (unlikely(p == RETRY_TASK))
                goto again;
            return p;
        }
    }

    BUG(); /* the idle class will always have a runnable task */
}
````

进程中所有的**调度器类**, 是通过**next域链接域链接**在一起的, 调度的顺序为

```c
stop -> dl -> rt -> fair -> idle
``` 

其中**for\_each\_class**遍历所有的调度器类,依次执行pick\_next\_task操作选择最优的进程

它会从优先级最高的sched\_class\_highest(目前是stop\_sched\_class)查起,依次按照调度器类的优先级从高到低的顺序调用调度器类对应的pick\_next\_task\_fair函数直到查找到一个能够被调度的进程

for\_each\_class定义在[kernel/sched/sched.h](http://lxr.free-electrons.com/source/kernel/sched/sched.h?v=4.6#L1251), 如下所示

```c
#define sched_class_highest (&stop_sched_class)
#define for_each_class(class) \
   for (class = sched_class_highest; class; class = class->next)

extern const struct sched_class stop_sched_class;
extern const struct sched_class dl_sched_class;
extern const struct sched_class rt_sched_class;
extern const struct sched_class fair_sched_class;
extern const struct sched_class idle_sched_class;
```

除了**全局的pick\_next\_task函数**,每个调度器类都提供了pick\_next\_task函数用以查找对应调度器下的最优进程, 其定义如下所示

| 调度器类 | pick\_next策略 | pick\_next\_task\_fair函数 |
| ------- |:-------|:-------|
| stop\_sched\_class |  | [kernel/sched/stop_task.c, line 121, pick\_next\_task\_stop](http://lxr.free-electrons.com/source/kernel/sched/stop_task.c?v=4.6#L27)|
| dl\_sched\_class | | [kernel/sched/deadline.c, line 1782, pick\_next\_task\_dl](http://lxr.free-electrons.com/source/kernel/sched/deadline.c?v=4.6#L1128)|
| rt\_sched\_class | **取出合适的进程**后, **dequeue\_pushable\_task**从**pushable队列**里取出来 | [/kernel/sched/rt.c, line 1508, pick\_next\_task\_rt](http://lxr.free-electrons.com/source/kernel/sched/rt.c?v=4.6#L1508)|
| fail\_sched\_class | **pick\_next\_task\_fair**,从**红黑树**里，选出vtime最小的那个进程，**调用set\_next\_entity将其出队** | [kernel/sched/fair.c, line 5441, pick\_next\_task\_fail](http://lxr.free-electrons.com/source/kernel/sched/fair.c?v=4.6#L5441) |
| idle\_sched\_class | **直接调度idle进程** | [kernel/sched/idle_task.c, line 26, pick\_next\_task\_idle](http://lxr.free-electrons.com/source/kernel/sched/idle_task.c?v=4.6#L26) |

>实际上，对于**RT进程**，**put和pick并不操作运行队列**
>
>对于FIFO和RR的区别，在scheduler\_tick中通过curr->sched\_class->task\_tick进入到task\_tick\_rt的处理,如果是**非RR的进程则直接返回**，否则**递减时间片，如果时间片耗完，则需要将当前进程放到运行队列的末尾**,这个时候才**操作运行队列**（**FIFO和RR进程，是否位于同一个plist队列**？），时间片到点，会**重新移动当前进程requeue\_task\_rt**，进程会被**加到队列尾**，接下来**set\_tsk\_need\_resched触发调度**，进程被抢占进入schedule

**问题1 : 为什么要多此一举判断所有的进程是否全是cfs调度的普通非实时进程?**

**加快经常性事件**, 是程序开发中一个优化的准则, 那么linux系统中最普遍的进程是什么呢? 肯定是非实时进程啊, 其调度器必然是cfs, 因此

```c
rev->sched_class == class && rq->nr_running == rq->cfs.h_nr_running
```

这种情形发生的概率是很大的, 也就是说**多数情形下**, 我们的linux中**进程全是cfs调度**的

而**likely这个宏**也表明了这点,这也是**gcc内建的一个编译选项**,它其实就是告诉编译器**表达式很大的情况下为真**, 编译器可以对此做出**优化**

```c
//  http://lxr.free-electrons.com/source/tools/virtio/linux/kernel.h?v=4.6#L91
 #ifndef likely
 # define likely(x)     (__builtin_expect(!!(x), 1))
 #endif
 
 #ifndef unlikely
 # define unlikely(x)   (__builtin_expect(!!(x), 0))
 #endif
```

## 2.4 context\_switch进程上下文切换

>进程上下文的切换其实是一个很复杂的过程, 我们在这里不能详述, 但是我会尽可能说明白
>
>具体的内容请参照

### 2.4.1 进程上下文切换

**上下文切换**(有时也称做**进程切换**或**任务切换**)是指CPU从一个进程或线程切换到另一个进程或线程

稍微详细描述一下，上下文切换可以认为是内核（操作系统的核心）在 CPU 上对于进程（包括线程）进行以下的活动：

1. 挂起一个进程，将这个**进程在 CPU 中的状态（上下文**）存储于**内存中的某处**，

2. 在内存中检索**下一个进程的上下文**并将其**在 CPU 的寄存器中恢复**

3. 跳转到**程序计数器所指向的位置**（即跳转到进程被中断时的代码行），以恢复该进程

因此上下文是指某一时间点**CPU寄存器**和**程序计数器**的内容,广义上还包括**内存中进程的虚拟地址映射信息**.

**上下文切换只能发生在内核态(！！！**)中,上下文切换通常是**计算密集型**的。也就是说，它需要相当可观的处理器时间，在每秒几十上百次的切换中，每次切换都需要纳秒量级的时间。所以，**上下文切换**对系统来说意味着**消耗大量的CPU时间**，事实上，可能是操作系统中**时间消耗最大的操作**。

Linux相比与其他操作系统（包括其他类Unix系统）有很多的优点，其中有一项就是，其**上下文切换**和**模式切换**的**时间消耗非常少**.

### 2.4.2 context\_switch流程

context\_switch函数完成了进程上下文的切换, 其定义在[kernel/sched/core.c]

context\_switch( )函数**建立next进程的地址空间**。进程描述符的**active\_mm**字段指向进程所使用的**内存描述符**，而**mm字段**指向进程所拥有的**用户空间内存描述符**。对于**一般的进程**，这**两个字段有相同的地址**，但是，**内核线程**没有它自己的**地址空间**而且它的**mm字段总是被设置为 NULL;active\_mm成员被初始化为前一个运行进程的active\_mm值,如果当前内核线程被调度之前运行的也是另外一个内核线程时候，那么其mm和avtive\_mm都是NULL**.

context\_switch( )函数保证：**如果next是一个内核线程**, 它**使用prev所使用的地址空间(！！！**)

它主要执行如下操作

- 调用**switch\_mm**(), 把**虚拟内存**从**一个进程**映射切换到**新进程**中

- 调用**switch\_to**(), 从上一个进程的**处理器状态**切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息

由于不同架构下地址映射的机制有所区别,而寄存器等信息弊病也是依赖于架构的,因此**switch\_mm**和**switch\_to**两个函数均是**体系结构相关**的

### 2.4.3 switch\_mm切换进程虚拟地址空间

switch\_mm主要完成了进程prev到next虚拟地址空间的映射, 由于**内核虚拟地址空间**是**不许切换**的, 因此切换的主要是**用户态的虚拟地址空间**

这个是一个****体系结构相关的函数****,其实现在对应体系结构下的[arch/对应体系结构/include/asm/mmu_context.h](http://lxr.free-electrons.com/ident?v=4.6;i=switch_mm)文件中, 我们下面列出了几个常见体系结构的实现

| 体系结构 | switch\_mm实现 |
| ------- |:-------|
| x86 | [arch/x86/include/asm/mmu\_context.h, line 118](http://lxr.free-electrons.com/source/arch/x86/include/asm/mmu_context.h?v=4.6#L118) |
| arm | [arch/arm/include/asm/mmu\_context.h, line 126](http://lxr.free-electrons.com/source/arch/arm/include/asm/mmu_context.h?v=4.6#L126) |
| arm64 | [arch/arm64/include/asm/mmu\_context.h, line 183](http://lxr.free-electrons.com/source/arch/arm64/include/asm/mmu_context.h?v=4.6#L183)

其主要工作就是**切换了进程的CR3**

>控制寄存器（CR0～CR3）用于控制和确定处理器的操作模式以及当前执行任务的特性
>
>CR0中含有控制**处理器操作模式**和**状态的系统控制标志**；
>
>CR1保留不用；
>
>CR2含有导致**页错误的线性地址**；
>
>CR3中含有**页目录表物理内存基地址**，因此该寄存器也被称为页目录基地址寄存器PDBR（Page-Directory Base address Register）。

### 2.4.4 switch\_to切换进程堆栈和寄存器

执行**环境的切换**是在**switch\_to**()中完成的,switch\_to**完成最终的进程切换**，它保存原进程的所有寄存器信息，恢复新进程的所有寄存器信息，并执行新的进程

**调度过程**可能**选择了一个新的进程**,而**清理工作**则是针对**此前的活动进程**,请注意,这**不是发起上下文切换的那个进程(！！！**),而是系统中随机的**某个其他进程**,内核必须想办法使得**进程**能够与**context\_switch例程(！！！**)通信,这就可以通过switch\_to宏实现.因此switch\_to函数通过3个参数提供2个变量, 

在**新进程被选中**时, **底层的进程切换例程**必须将**此前执行的进程**提供给**context\_switch例程**,由于**控制流会回到该函数的中间(！！！**),这无法用普通的函数返回值来做到, 因此提供了**3个参数的宏 **


```c
/*
 * Saving eflags is important. It switches not only IOPL between tasks,
 * it also protects other tasks from NT leaking through sysenter etc.
*/
#define switch_to(prev, next, last)
```

| 体系结构 | switch\_to实现 |
| ------- |:-------:|
| x86 | arch/x86/include/asm/switch\_to.h中两种实现<br><br> [定义CONFIG\_X86\_32宏](http://lxr.free-electrons.com/source/arch/x86/include/asm/switch_to.h?v=4.6#L27)<br><br>[未定义CONFIG\_X86\_32宏](http://lxr.free-electrons.com/source/arch/x86/include/asm/switch_to.h?v=4.6#L103) |
| arm | [arch/arm/include/asm/switch_to.h, line 25](http://lxr.free-electrons.com/source/arch/arm/include/asm/switch_to.h?v=4.6#L18) |
| 通用 | [include/asm-generic/switch_to.h, line 25](http://lxr.free-electrons.com/source/include/asm-generic/switch_to.h?v=4.6#L25) |

内核在switch\_to中执行如下操作

1. 进程切换, 即**esp的切换**, 由于**从esp可以找到进程的描述符**

2. **硬件上下文切换**, 设置ip寄存器的值, 并**jmp到\_\_switch\_to函数**

3. **堆栈的切换**, 即ebp的切换, ebp是栈底指针, 它确定了**当前用户空间**属于哪个进程

## 2.5 need\_resched, TIF\_NEED\_RESCHED标识与用户抢占

### 2.5.1 need\_resched标识TIF\_NEED\_RESCHED

内核在**即将返回用户空间**时**检查进程是否需要重新调度**，如果设置了，就会发生调度, 这被称为**用户抢占**, 因此**内核在thread\_info的flag中设置了一个标识来标志进程是否需要重新调度,即重新调度need\_resched标识TIF\_NEED\_RESCHED**

并提供了一些设置可检测的函数

| 函数 | 描述 | 定义 |
| ------- |:-------:|:-------:|
| set\_tsk\_need\_resched | 设置指定进程中的need\_resched标志 | [include/linux/sched.h, L2920](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.6#L2920) |
| clear\_tsk\_need\_resched | 清除指定进程中的need\_resched标志 | [include/linux/sched.h, L2926](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.6#L2931) |
| test\_tsk\_need\_resched | 检查指定进程need\_resched标志 | [include/linux/sched.h, L2931](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.6#L2931) |

而我们内核中调度时常用的**need\_resched**()函数**检查进程是否需要被重新调度**其实就是通过**test\_tsk\_need\_resched**实现的, 其定义如下所示

```c
// http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.6#L3093
static __always_inline bool need_resched(void)
{
	return unlikely(tif_need_resched());
}

// http://lxr.free-electrons.com/source/include/linux/thread_info.h?v=4.6#L106
#define tif_need_resched() test_thread_flag(TIF_NEED_RESCHED)
```

### 2.5.2 用户抢占和内核抢占

当内核即将返回用户空间时,内核会检查need\_resched是否设置，如果设置，则**调用schedule**()，此时，发生**用户抢占**。

一般来说，用户抢占发生几下情况

1. 从系统调用返回用户空间

2. 从中断(异常)处理程序返回用户空间

当kernel(**系统调用或者中断都在kernel中！！！**)返回用户态时，系统可以安全的执行当前的任务，或者切换到另外一个任务.

当中断处理例程或者系统调用完成后, kernel返回用户态时, need\_resched标志的值会被检查, 假如它为1, **调度器**会选择一个**新的任务并执行**. **中断和系统调用的返回路径**(return path)的实现在**entry.S**中(entry.S不仅包括**kernel entry code**，也包括**kernel exit code**)。

抢占时伴随着schedule()的执行,因此内核提供了一个TIF\_NEED\_RESCHED标志来表明是否要用schedule()调度一次

根据抢占发生的时机分为用户抢占和内核抢占。

**用户抢占**发生在**内核即将返回到用户空间**的时候。**内核抢占**发生在****返回内核空间****的时候。

| 抢占类型 | 描述 | 抢占发生时机 |
| ------- |:-------:|:-------:|
| 用户抢占 | 内核在即将返回用户空间时检查进程是否设置了TIF\_NEED\_RESCHED标志，如果设置了，就会发生用户抢占.  |  从系统调用或中断处理程序返回用户空间的时候 |
| 内核抢占 | 在不支持内核抢占的内核中，内核进程如果自己不主动停止，就会一直的运行下去。无法响应实时进程. 抢占内核虽然牺牲了上下文切换的开销, 但获得 了更大的吞吐量和响应时间<br><br>2.6的内核添加了内核抢占，同时为了某些地方不被抢占，又添加了自旋锁. 在进程的thread\_info结构中添加了preempt\_count该数值为0，当进程使用一个自旋锁时就加1，释放一个自旋锁时就减1. 为0时表示内核可以抢占. | 1.	从中断处理程序返回内核空间时，内核会检查preempt\_count和TIF\_NEED\_RESCHED标志，如果进程设置了 TIF\_NEED\_RESCHED标志,并且preempt\_count为0，发生内核抢占<br><br>2.	当内核再次用于可抢占性的时候，当进程所有的自旋锁都释 放了，释放程序会检查TIF\_NEED\_RESCHED标志，如果设置了就会调用schedule<br><br>3.	显示调用schedule时<br><br>4.	内核中的进程被堵塞的时候 |

# 3 总结

## 3.1 **schedule调度流程**

schedule就是**主调度器的函数**, 在内核中的许多地方,如果要将CPU分配给与当前活动进程不同的另一个进程, 都会直接调用主调度器函数schedule, 该函数定义在[kernel/sched/core.c, L3243](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.6#L3243), 如下所示

该函数完成如下工作

1. **确定当前就绪队列**, 并在保存一个指向当前(仍然)活动进程的task\_struct指针

2. **检查死锁, 关闭内核抢占**后调用\_\_schedule完成内核调度

3. 恢复内核抢占, 然后检查当前进程是否设置了重调度标志TLF\_NEDD\_RESCHED,如果该进程被其他进程设置了TIF\_NEED\_RESCHED标志, 则函数重新执行进行调度

```c
    do {
        preempt_disable();									/*  关闭内核抢占  */
        __schedule(false);									/*  完成调度  */
        sched_preempt_enable_no_resched();	                /*  开启内核抢占  */
    } while (need_resched());	/* 如果该进程被其他进程设置了TIF_NEED_RESCHED标志，则函数重新执行进行调度 */
```

## 3.2 **\_\_schedule如何完成内核抢占**

1. 完成一些必要的检查, 并设置进程状态, 处理**进程所在的就绪队列**

2. 调度全局的pick\_next\_task选择抢占的进程

	- 如果当前cpu上所有的进程都是cfs调度的普通非实时进程, 则直接用cfs调度, 如果无程序可调度则调度idle进程

	- 否则从优先级最高的调度器类sched\_class\_highest(目前是stop\_sched\_class)开始依次遍历所有调度器类的pick\_next\_task函数, 选择最优的那个进程执行

3. context\_switch完成进程上下文切换
	
	- 调用switch\_mm(), 把虚拟内存从一个进程映射切换到新进程中

	- 调用switch\_to(),从上一个进程的处理器状态切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息

## 3.3 **调度的内核抢占和用户抢占**

内核在完成调度的过程中总是先关闭内核抢占, 等待内核完成调度的工作后, 再把内核抢占开启, 如果在内核完成调度器过程中, 这时候如果发生了内核抢占, 我们的调度会被中断, 而调度却还没有完成, 这样会丢失我们调度的信息.

而同样我们可以看到, 在调度完成后, 内核会去判断need\_resched条件, 如果这个时候为真, 内核会重新进程一次调度, 此次调度由于发生在内核态因此仍然是一次内核抢占

need\_resched条件其实是判断**need\_resched标识TIF\_NEED\_RESCHED**的值, 内核在thread\_info的flag中设置了一个标识来标志进程是否需要重新调度, 即重新调度need\_resched标识TIF\_NEED\_RESCHED, 内核在即将返回用户空间时会检查标识TIF\_NEED\_RESCHED标志进程是否需要重新调度，如果设置了，就会发生调度, 这被称为**用户抢占**,

而内核抢占是通过**自旋锁preempt\_count**实现的,同样当内核可以进行内核抢占的时候(比如从中断处理程序返回内核空间或内核中的进程被堵塞的时候)，内核会检查preempt\_count和TIF\_NEED\_RESCHED标志，如果进程设置了 TIF\_NEED\_RESCHED标志,并且preempt\_count为0，发生**内核抢占**