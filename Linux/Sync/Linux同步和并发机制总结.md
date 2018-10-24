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