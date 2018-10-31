[TOC]

# 1 原子操作

原子操作保证**指令以原子的方式执行**(不是代码编写, 而是由于CPU的行为导致, 先加载变量到本地寄存器再操作再写内存). **x86**的**atomic操作**通常通过**原子指令**或**lock前缀**实现.

# 2 内存屏障

内存屏障是程序在运行时的实际内存访问顺序和程序代码编写的访问顺序不一致，会导致内存乱序访问。分为**编译时编译器优化**和**运行时多CPU间交互**.

- 编译优化使用**barrier**()**防止编译优化**

```c
[include/linux/compiler-gcc.h]
#define barrier() __asm__ __volatile__ ("" ::: "memory")
```

"memory"强制**gcc编译器**假设RAM**所有内存单元均被汇编指令修改**，这样**cpu中的registers和cache**中已缓存的内存单元中的**数据将作废(！！！**)。**cpu**将不得不在需要的时候**重新读取内存中的数据**。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。目的是**防止编译乱序(！！！**)。

- x86提供了三个指令: sfence(该指令前的写操作必须在该指令后的写操作前完成), lfence(读操作), mfence(读写操作)

# 3 spinlock

操作系统中**锁的机制分为两类(！！！**)，一类是**忙等待**，另一类是**睡眠等待**。

**spinlock**主要针对**数据操作集合的临界区**, 临界区是**一个变量**, **原子变量**可以解决. 抢到锁的**进程不能睡眠**, 并要**快速执行**(这是spinlock的**设计语义**).

"FIFO ticket\-based算法": 

锁由slock(和\<next, owner\>组成union)构成

- 获取锁: CPU先领ticket(当前next值), 然后锁的next加1, owner等于ticket, CPU获得锁, 返回, 否则循环忙等待
- 释放锁: 锁的owner加1

获取锁接口(这里只提这两个):

- spin\_lock(): 关内核抢占, 循环抢锁, 但来中断(中断会抢占所有)可能导致死锁
- spin\_lock\_irq(): 关本地中断, 关内核抢占(内核不会主动抢占), 循环抢锁

spin\_lock()和raw\_spin\_lock():

- 在绝对不允许被抢占和睡眠的临界区，使用raw\_spin\_lock
- Linux实时补丁spin\_lock()允许**临界区抢占**和**进程睡眠**, 否则spin\_lock()直接调用raw\_spin\_lock()

spin\_lock特点:

- 忙等待, 不断尝试
- **同一时刻只能有一个获得**
- spinlock临界区尽快完成, 不能睡眠
- 可以在中断上下文使用

# 4 信号量semaphore

信号量**允许进程进入睡眠状态(即睡眠等待**), 是计数器, 支持两个操作原语P(down)和V(up)

```c
struct semaphore{
    raw_spinlock_t		lock;       //对count和wait_list的保护
	unsigned int		count;      // 允许持锁数目
	struct list_head	wait_list;  // 没成功获锁的睡眠的进程链表
};
```

初始化: sema\_init()

获得锁:

- down(struct semaphore \*sem): 失败进入**不可中断的睡眠**状态
- down\_interruptible(struct semaphore \*sem): **失败**则进入**可中断的睡眠**状态. ①关闭**本地中断(防止中断来导致死锁**); ②count大于0, 当前进程获得semaphore, count减1, 退出; ③count小于等于0, 将**当前进程加入wait\_list链表**, 循环: 设置**进程TASKINTERRUPTIBLE**, 调用**schedule\_timeout()让出CPU<即睡眠**>, 判断被调度到的原因(**能走到这儿说明又被调度**到了), 如果waiter.up为true, 说明**被up操作唤醒**, 获得信号量,退出; ④打开本地中断
- 等等

释放锁:

up(struct semaphore \*sem): wait\_list为空, 说明没有进程等待信号量, count加1, 退出; wait\_list不为空, 等待队列是先进先出, 将第一个移出队列, 设置waiter.up为true, wake\_up\_process()唤醒waiter\-\>task进程

**睡眠等待, 任意数量的锁持有者**

# 5 Mutex和MCS锁

## 5.1 MCS锁

传统自旋等待锁, 在多CPU和NUMA系统, 所有自旋等待锁在同一个共享变量自旋, 对同一个变量修改, 由cache一致性原理(例如MESI)导致参与自旋的CPU中的cache line变的无效, 从而导致CPU高速缓存行颠簸现象(CPU cache line bouncing), 即多个CPU上的cache line反复失效.

MCS减少CPU cache line bouncing, 核心思想是每个锁的申请者只在本地CPU的变量上自旋，而不是全局的变量。

```c
[include/linux/osq_lock.h]
// Per-CPU的, 表示本地CPU节点
struct optimistic_spin_node {
	struct optimistic_spin_node *next, *prev; // 双向链表
	int locked; /* 1 if lock acquired */ // 加锁状态, 1表明当前能申请
	int cpu; /* encoded CPU # + 1 value */ // CPU编号, 0表示没有CPU, 1表示CPU0, 类推
};
// 一个MCS锁一个
struct optimistic_spin_queue {
	/*
	 * Stores an encoded value of the CPU # of the tail node in the queue.
	 * If the queue is empty, then it's set to OSQ_UNLOCKED_VAL.
	 */
	atomic_t tail;
};
```

初始化: osq\_lock\_init()

申请MCS锁: osq\_lock()

- 给当前进程所在CPU编号, 将lock\-\>tail设为当前CPU编号, 如果原有tail是0, 表明没有CPU, 那直接申请成功
- 通过原有tail获得前继node, 然后将当前node节点加入MCS链表
- 循环自旋等待, 判断当前node的locked是否是1, 1的话说明前继释放了锁, 申请成功, 退出; 不是1, 判断是否需要重新调度(抢占或调度器要求), 是的话放弃自旋等待, 退出MCS链表, 删除MCS链表节点, 申请失败

![config](./images/1.png)

释放MCS锁: osq\_unlock()

- lock\-\>tail是当前CPU, 说明没人竞争, 直接设tail为0, 退出
- 当前节点的后继节点存在, 设置后继node的locked为1, 相当于传递锁

## 5.2 Mutex锁

```c
[include/linux/mutex.h]
struct mutex {
	/* 1: unlocked, 0: locked, negative: locked, possible waiters */
	// 1表示没人持有锁；0表示锁被持有；负数表示锁被持有且有人在等待队列中等待
	atomic_t		count; 
	// 保护wait_list睡眠等待队列
	spinlock_t		wait_lock;
	// 所有在该Mutex上睡眠的进程
	struct list_head	wait_list;
#if defined(CONFIG_MUTEX_SPIN_ON_OWNER)
    // 锁持有者的task_struct
	struct task_struct	*owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    // MCS锁
	struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
};
```

初始化: 静态DEFINE\_MUTEX宏, 动态使用mutex\_init()函数

申请Mutex锁:

- mutex\-\>count减1等于0, 说明没人持有锁, 直接申请成功, 设置owner为当前进程, 退出
- 申请OSQ锁, 减少CPU cache line bouncing, 会将所有等待Mutex的参与者放入OSQ锁队列, 只有第一个等待者才参与自旋等待
- while循环自旋等待锁持有者释放, 这中间①锁持有者变化 ②锁持有进程被调度出去, 即睡眠(task\-\>on\_cpu=0), ③调度器需要调度其他进程(need\_resched())都会**退出循环**, 但**不是锁持有者释放了锁(lock\-\>owner不是NULL**); 如果是锁持有者释放了锁(lock\-\>owner是NULL), 当前进程获取锁, 设置count为0, 释放OSQ锁, 申请成功, 退出. 
- 上面自旋等待获取锁失败, 再尝试一次申请, 不成功的话只能走睡眠唤醒的慢车道. 
- 将当前进程的waiter进入mutex等待队列wait\_list
- 循环: 获取锁, 失败则让出CPU, 进入睡眠态, 成功则退出循环, 收到异常信号也会退出循环
- 成功获取锁退出循环的话, 设置进程运行态, 从等待队列出列, 如果等待队列为空, 设置lock\-\>count为0(表明锁被人持有且队列无人)
- 设置owner为当前进程

释放Mutex锁:

- 清除owner
- count加1若大于0, 说明队列没人, 成功
- 释放锁, 将count设为1, 然后唤醒队列第一个进程(waiter\-\>task)

从 **Mutex**实现细节的分析可以知道，**Mutex**比**信号量**的实现要**高效很多**。

- Mutex**最先**实现**自旋等待机制**。
- Mutex在**睡眠之前尝试获取锁(！！！**)。
- Mutex实现**MCS锁**来**避免多个CPU争用锁**而导致**CPU高速缓存行颠簸**现象。

正是因为**Mutex**的**简洁性**和**高效性**，因此**Mutex的使用场景**比**信号量**要**更严格**，使用Mutex需要注意的**约束条件**如下。

- **同一时刻**只有**一个线程**可以持有Mutex。
- 只有**锁持有者可以解锁**。不能在一个进程中持有Mutex，而在另外一个进程中释放它。因此Mutex**不适合内核同用户空间复杂的同步场景(！！！**)，信号量和读写信号量比较适合。
- **不允许递归地加锁和解锁**。
- 当**进程持有Mutex**时，进程**不可以退出(！！！**)。
- Mutex必须使用**官方API来初始化**。
- Mutex可以**睡眠**，所以**不允许**在**中断处理程序**或者**中断下半部**中使用，例如tasklet、定时器等。

在实际工程项目中，该**如何选择spinlock、信号量和Mutex**呢？

在**中断上下文**中毫不犹豫地使用**spinlock**, 如果**临界区**有**睡眠、隐含睡眠的动作**及**内核API**，应**避免选择spinlock**。在**信号量**和**Mutex**中该如何选择呢？除非代码场景**不符合上述Mutex的约束**中有某一条，**否则都优先使用Mutex**。

# 6 读写锁

**信号量缺点**: 没有区分临界区的**读写属性**

读写锁特点:

- 允许**多个读者**同时进入临界区，但**同一时刻写者不能进入**。
- **同一时刻**只允许**一个写者**进入临界区。
- **读者和写者不能同时进入临界区**。

读写锁有**两种**，分别是**spinlock类型**和**信号量类型**。分别对应**typedef struct rwlock\_t**和**struct rw\_semaphore**.

**读写锁**在内核中应用广泛，特别是在**内存管理**中，全局的**mm\->mmap\_sem读写信号量**用于保护进程地址空间的一个读写信号量，还有**反向映射RMAP系统中的anon\_vma\->rwsem**，地址空间**address\_space**数据结构中**i\_mmap\_rwsem**等。

再次总结**读写锁的重要特性**。

- **down\_read**(): 如果一个进程持有了**读者锁**，那么允许继续申请**多个读者锁**，申请**写者锁**则要**睡眠等待**。
- **down\_write**(): 如果一个进程持有了**写者锁**，那么第二个进程申请该**写者锁**要**自旋等待(配置了CONFIG\_RWSEM\_SPIN\_ON\_OWNER选项不睡眠**)，申请**读者锁**则要**睡眠等待**。
- **up\_write**()/**up\_read**():如果**等待队列**中**第一个成员是写者**，那么**唤醒该写者**，否则**唤醒排在等待队列**中**最前面连续的几个读者**。

# 7 RCU

## 7.1 背景和原理

spinlock、读写信号量和mutex的实现，它们都使用了原子操作指令，即原子地访问内存，多CPU争用共享的变量会让cache一致性变得很糟(！！！)，使得性能下降。读写信号量还有一个致命缺点, 只允许多个读者同时存在, 但读者和写者不能同时存在.

RCU实现目标是, 读者线程没有同步开销(不需要额外的锁, 不需要使用原子操作指令和内存屏障); 同步任务交给写者线程, 写者线程等所有读者线程完成把旧数据销毁, 多个写者则需要额外保护机制.

原理: RCU记录所有指向共享数据的指针的使用者, 当修改该共享数据时，首先创建一个副本，在副本中修改。所有读访问线程都离开读临界区之后，指针指向新的修改后副本的指针，并且删除旧数据。

## 7.2 操作接口

接口如下:

- rcu\_read\_lock()/rcu\_read\_unlock(): 组成一个**RCU读临界**。
- rcu\_dereference(): 用于获取**被RCU保护的指针**(RCU protected pointer)，**读者线程**要访问**RCU保护的共享数据**，需要使用**该函数创建一个新指针(！！！**)，并且**指向RCU被保护的指针**。
- rcu\_assign\_pointer(): 通常用在**写者线程**。在**写者线程**完成新数据的**修改**后，调用该接口可以让**被RCU保护的指针**指向**新创建的数据**，用RCU的术语是发布(Publish) 了更新后的数据。
- synchronize\_rcu(): 同步等待**所有现存的读访问完成**。
- call\_rcu(): 注册一个**回调函数(！！！**)，当**所有**现存的**读访问完成**后，调用这个回调函数**销毁旧数据**。

```c
[RCU的一个简单例子]
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/slab.h>
#include <linux/spinlock.h>
#include <linux/rcupdate.h>
#include <linux/kthread.h>
#include <linux/delay.h>

struct foo {
	int a;
	struct rcu_head rcu;
};
        
static struct foo *g_ptr;

static void myrcu_reader_thread(void *data) //读者线程
{
    struct foo *p = NULL;
    
    while(1){
        msleep(200);
        // 重点1
        rcu_read_lock();
        // 重点2
        p = rcu_dereference(g_ptr);
        if(p)
            printk("%s: read a=%d\n", __func__, p->a);
        // 重点3
        rcu_read_unlock();
    }
}

static void myrcu_del(struct rcu_head *rh)
{
    struct foo *p = container_of(rh, struct foo, rcu);
    printk ("%s: a=%d\n", __func__, p->a);
    kfree(p);
}

static void myrcu_writer_thread(void *p)    //写者线程
{
    struct foo *mew;
    struct foo *old;
    int value = (unsigned long)p;
    
    while(1){
        msleep(400);
        struct foo *new_ptr = kmalloc(sizeof(struct foo), GFP_KERNEL);
        old = g_ptr;
        printk("%s: write to new %d\n", __func__, value);
        *new_ptr = *old;
        new_ptr->a = value;
        // 重点
        rcu_assign_pointer(g_ptr, new_ptr);
        // 重点
        call_rcu(&old->rcu, myrcu_del);
        value++;
    }
}

static int __init my_test_init(void){
    struct task_struct *reader_thread;
    struct task_struct *writer_thread ；
    int value = 5;
    
    printk("figo: my module init\n");
    g_ptr = kzalloc(sizeof (struct foo), GFP_KERNEL);
    
    reader_thread = kthread_run(myrcu_reader_thread, NULL, "rcu_reader");
    writer_thread = kthread_run(myrcu_writer_thread, (void *)(unsigned long)value, "rcu_writer")
    return 0;
}

static void __exit my_test_exit(void)
{
    printk("goodbye\n");
    if(g_ptr)
        kfree(g_ptr);
}
MODULE_LICENSE("GPL");
module_init(my_test_init);
```

该例子的目的是通过RCU机制保护**my\_test\_init**()**分配的共享数据结构g\_ptr**，另外创建了一个读者线程和一个写者线程来模拟同步场景。

对于**读者线程**myrcu\_reader\_thread:

- 通过**rcu\_read\_lock**()和**rcu\_read\_unlock**()来构建一个**读者临界区**。
- 调用rcu\_dereference()获取**被保护数据g\_ptr指针**的一个**副本(！！！**)，即**指针p**，这时**p和g\_ptr**都指向**旧的被保护数据**。
- 读者线程每隔200毫秒读取一次被保护数据。

对于**写者线程**myrcu\_writer\_thread:

- 分配一个**新的保护数据new\_ptr**，并修改相应数据。
- **rcu\_assign\_pointer**()让g\_ptr指向**新数据**。
- call\_rcu()注册一个**回调函数(！！！**)，确保**所有对旧数据的引用都执行完成**之后，才调用回调函数来删除旧数据old\_data。
- 写者线程每隔400毫秒修改被保护数据。

在所有的**读访问完成**之后，内核可以释放旧数据，对于**何时释放旧数据**，内核提供了**两个API函数(！！！**)：**synchronize\_rcu**()和 **call\_rcu**().

## 7.3 基本概念

Grace Period, 宽限期: **GP开始**到**所有读者临界区的CPU离开**算一个GP, GP结束调用回调函数

Quiescent State, 静止状态: 一个CPU处于读者临界区, 说明活跃, 离开读者临界区, 静止态

经典RCU使用全局cpumask位图, 每个比特一个CPU. 每个CPU在GP开始设置对应比特, 结束清相应比特. 多CPU会竞争使用, 需要使用spinlock, CPU越多竞争越惨烈.

Tree RCU解决cpumask位图竞争问题.

![config](./images/2.png)

以4核处理器为例，假设Tree RCU把**两个CPU**分成**1个rcu\_node节点**，这样**4个CPU**被分配到**两个rcu\_node**节点上，另外还有**1个根rcu\_node**节点来**管理这两个rcu\_node节点**。如图4.8所示，**节点1**管理**cpuO**和**cpul**，节点2管理cpu2和cpu3，而节点0是根节点，管理节点1和节点2。**每个节点**只需要**两个比特位的位图(！！！**)就可以**管理各自的CPU**或者节点，**每个节点(！！！**)都有**各自的spinlock锁(！！！**)来**保护相应的位图**。

注意: **CPU不算层次level！！！**

假设**4个CPU**都经历过**一个QS状态**，那么**4个CPU**首先在**Level 0层级**的**节点1**和**节点2**上**修改位图**。对于**节点1**或者**节点2**来说，**只有两个CPU来竞争锁**，这比经典RCU上的锁争用要减少一半。当**Level 0**上**节点1**和**节点2**上**位图都被清除干净后(！！！**)，才会清除**上一级节点的位图**，并且**只有最后清除节点的CPU(！！！**)才有机会去**尝试清除上一级节点的位图(！！！**)。因此对于节点0来说，还是两个CPU来争用锁。整个过程都是只有两个CPU去争用一个锁，比经典RCU实现要减少一半。

## 7.4 Linux实现

### 7.4.1 相关数据结构定义

Tree RCU实现, 定义了3个很重要的数据结构，分别是struct rcu\_data、struct rcu\_node和struct rcu\_state，另外还维护了一个比较隐晦的状态机(！！！).

- struct rcu\_data数据结构定义成Per\-CPU变量. gpnum和completed用于GP状态机的运行状态, 初始两个都等于\-300, 新建一个GP, gpnum加1; GP完成, completed加1. passed\_quiesce(bool): 当时钟tick检测到rcu\_data对应的CPU完成一次Quiescent State, 设这个为true. qs\_pending(bool): CPU正等待QS.
- struct rcu\_node是Tree RCU中的组成节点，它有根节点(Root Node)和叶节点之分。如果Tree RCU只有一层，那么根节点下面直接管理着一个或多个rcu\_data;如果Tree RCU有多层结构，那么根节点管理着多个叶节点，**最底层的叶节点**管理者**一个或多个rcu\_data**。
- RCU系统支持多个不同类型的RCU状态，使用struct rcu\_state数据结构来描述这些状态。每种RCU类型都有独立的层次结构(！！！)，即根节点和rcu\_data数据结构。也有gpnum和completed.

Tree通过三个维度确定层次关系: **每个叶子的CPU数量(CONFIG\_RCU\_FANOUT\_LEAF**), 每层的最多叶子数量(CONFIG\_RCU\_FANOUT), 最多层数(MAX\_RCU\_LVLS宏定义, 是4, CPU不算level层次！！！)

![config](./images/5.png)

在**32核处理器**中，层次结构分成两层，**Level 0**包括**两个struct rcu\_node(！！！**)，其中**每个struct rcu\_node**管理**16个struct rcu\_data(！！！**)数据结构，分别表示**16个CPU的独立struct rcu\_data**数据结构; 在**Level 1**层级，有**一个struct rcu\_node(！！！**)节点**管理**着**Level 0层级**的**两个rcu\_node**节点，**Level 1**层级中的**rcu\_node**节点称为**根节点**，**Level 0**层级的**两个rcu\_node**节点是**叶节点**。

### 7.4.2 内核启动进行RCU初始化

![config](./images/3.png)

(1) 初始化3个rcu\_state, rcu\_sched\_state(普通进程上下文的RCU)、rcu\_bh\_state(软中断上下文)和rcu\_preempt\_state(系统配置了CONFIG\_PREEMPT\_RCU, 默认使用这个)

(2) 注册一个SoftIRQ回调函数

(3) 初始化每个rcu\_state的层次结构和相应的Per\-CPU变量rcu\_data

(4) 为每个rcu\_state初始化一个内核线程, 以rcu\_state命名

### 7.4.3 开启一个GP

1. **写者程序注册RCU回调函数**:

- 参数: rcu_head(每个RCU保护的数据都会内嵌一个), 回调函数指针(GP结束<读者执行完>, 被调用销毁)

- 将rcu\_head加入到本地rcu\_data的nxttail链表

2. 每次**时钟中断处理函数tick\_periodic**(), 检查本地CPU的rcu\_data成员nxttail链表**有没有写者注册的回调函数**, 有的话**触发一个软中断raise\_softirq**().

3. **软中断处理函数**, 针对**每一个rcu\_state(！！！**): 检查rcu\_data成员nxttail链表**有没有写者注册的回调函数**, 有的话, 调整链表, 设置rsp\->gp\_flags标志位为RCU\_GP\_FLAG\_INIT, rcu\_gp\_kthread\_wake()唤醒**rcu\_state对应的内核线程(！！！**)

### 7.4.4 初始化一个GP

RCU内核线程真正初始化一个GP

### 7.4.5 检测QS

时钟中断处理函数判断当前CPU是否经过了一个quiescent state

### 7.4.6 回调函数

整个GP结束, RCU调用回调函数做一些销毁动作, 还是在RCU软中断中触发.

从代码中的**trace功能定义**的状态来看，**一个GP需要经历的状态转换**为: “**newreq \-\> start \-\> cpustart \-\> fqswait \-\> cpuend \-\>end**”。

总结Tree RCU的实现中有如下几点需要大家再仔细体会。

- Tree RCU为了避免**修改CPU位图带来的锁争用**，巧妙设置了树形的层次结构，**rcu\_data**、**rcu\_node**和**rcu\_state**这 3 个数据结构组成一棵完美的树。
- Tree RCU的实现维护了一个**状态机**，这个状态机若隐若现，只有把**trace功能打开**了才能感觉到该状态机的存在，trace函数是trace\_rcu\_grace\_period()。
- 维护了一些以rcu\_data\-\>nxttail\[\]二级指针为首的链表，该链表的实现很巧妙地运用了二级指针的指向功能。
- rcu\_data、rcu\_node和rcu\_state这3个数据结构中的gpnum、completed、grpmask、passed\_quiesce、qs\_pending、qsmask等成员，正是这些成员的值的变化推动了Tree RCU状态机的运转。

如图4.17所示是Tree RCU状态机的运转情况和一些重要数据的变化情况。

![config](./images/4.png)