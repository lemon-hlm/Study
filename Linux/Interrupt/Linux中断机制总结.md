[TOC]

# 1 Linux整体机制

当外设触发一次中断后，一个大概的处理过程是：

1、具体CPU architecture相关的模块会进行现场保护，然后调用machine driver(！！！这是platform driver<位于drivers/of/platform.c>??)对应的中断处理handler

2、machine driver对应的中断处理handler中会根据硬件的信息获取HW interrupt ID，并且通过irq domain模块翻译成IRQ number

3、调用该IRQ number对应的high level irq event handler，在这个high level的handler中，会通过和interupt controller交互，进行中断处理的flow control（处理中断的嵌套、抢占等），当然最终会遍历该中断描述符的IRQ action list，调用外设的specific handler来处理该中断

4、具体CPU architecture相关的模块会进行现场恢复。

## 1.1 硬件中断号和软件中断号映射

Linux维护了一个位图allocated\_irqs用来管理软中断号, 

系统初始化阶段会注册所有的中断控制器(包括虚拟的)irq\_domain(中断域), 并添加到全局链表irq\_domain\_list(通过irq\_domain的link成员), 如果线性映射的话, 初始化线性相关数据并根据当前控制器能支持的中断源的个数分配一个数组; 树存储, 初始化一个R树.

```c
[include/linux/irqdomain.h]

struct irq_domain {
	struct list_head link;
	const char *name;
	const struct irq_domain_ops *ops;
	void *host_data;
	unsigned int flags;

	/* Optional data */
	struct device_node *of_node;
	struct fwnode_handle *fwnode;
	enum irq_domain_bus_token bus_token;
	struct irq_domain_chip_generic *gc;
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	struct irq_domain *parent;
#endif

	/* reverse map data. The linear map gets appended to the irq_domain */
	irq_hw_number_t hwirq_max;
	unsigned int revmap_direct_max_irq;
	unsigned int revmap_size;
	struct radix_tree_root revmap_tree;
	unsigned int linear_revmap[];
};
```

- link: 用于将**irq domain**连接到**全局链表irq\_domain\_list**中。
- name： irq domain 的**名称**。
- ops: irq domain映射**操作使用的方法集合**。
- of\_node:对应**中断控制器的device node**。
- hwirq\_max: 该**irq domain**支持**中断数量的最大值**。
- revmap\_size: **线性映射的大小**。
- revmap\_tree: **Radix Tree 映射的根节点**。
- linear\_revmap:**线性映射用到的lookup table**。

系统初始化解析设备信息, 得到外设硬件中断号和触发类型, 通过全局allocated\_irqs位图分配一个空闲比特位(外设和中断控制器的request line有且仅有一条)从而获得一个IRQ号, 分配一个中断描述符irq\_desc, 配置CONFIG\_SPARE\_IRQ使用R树存储, 否则是全局数组, 将硬件中断号设置到中断描述符, 从而建立硬件中断号和软件中断号的映射关系.

## 1.2 注册中断

request\_irq()和线程化注册函数request\_threaded\_irq()

```c
[kernel/irq/manage.c]
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
```

- irq: IRQ中断号，注意这里使用的是**软件中断号**，而**不是硬件中断号**。
- handler: 指 **primary handler**, 有些类似于旧版本API函数request\_irq()的中断处理函数handler。中断发生时会**优先执行primary handler**。如果**primary handler为NULL**且**thread\_fn不为NULL**, 那么会执行系统**默认的 primary handler**: **irq\_default\_primary\_handler**()函数。
- thread\_fn: 中断**线程化的处理程序**。如果**thread\_fn不为NULL**, 那么会**创建一个内核线程**。primary handler和thread\_fn**不能同时为NULL**。
- irqflags: 中断**标志位**，如表5.2所示。
- devname: 该**中断名称**。
- dev\_id: 传递给**中断处理程序的参数**。

注册的不是中断号, 而是一个irqaction. 然后唤醒内核线程, 但线程会很快调用schedule(), 换出等待,

如果启用中断线程化, 则primary handler应该返回IRQ\_WAKE\_THREAD来唤醒中断线程。

(1) 共享中断必须传入dev\_id.

(2) 通过IRQ获取中断描述符(系统初始化已经分配好了). 

(3) primary handler和thread\_fn不能同时为NULL. primary为NULL, 设为默认的irq\_default\_primary\_handler.

(4) 分配一个irqaction(这是中断描述符的aciton链表中的当前设备对应的irqaction, 里面有primary handler和thread\_fn, 注意和中断描述符的handler\<high level event handler\>区分)

(5) 有thread\_fn的创建一个内核线程, 设为实时线程, SCHED\_FIFO调度策略, 优先级50, 线程名irq/中断号\-中断名, 执行函数为irq\_thread, 参数为当前irqaction. 增加线程usage计数, 确保线程异常不释放task\_struct, 防止中断线程访问空指针, 将当前设备中断的irqaction的thread指向它

(6) 将当前irqaction加到中断描述符的action链表

(7) 有thread\_fn, 则唤醒创建的内核线程, 该线程会很快睡眠, 详见下面线程执行

## 1.3 底层中断处理

具体看体系结构

## 1.4 高层中断处理

驱动读取硬件中断号, 调用内核代码继续处理

(1) irq\_enter(): 增加当前进程thread\_info的preempt\_count中的HARDIRQ域值, 表明进入硬件中断上下文

(2) irq\_find\_mapping()通过硬件中断号查找IRQ中断号(irq domain模块中R树或数组)

(3) 获得中断描述符, 调用desc\-\>handle\_irq(), 即high level event handler, 不同触发类型不同high level event handler. 基本操作都是中断响应相关, 然后遍历中断描述符的action链表, 依次执行每个action的primary handler, 一般而言, primary handler编写最开始要通过dev\_id确认是否是自己的设备产生了中断

(4) primary handler如果返回IRQ\_WAKE\_THREAD(默认primary就是直接返回这个),设置aciton的flags为IRQTF\_RUNTHREAD(已经置位说明已经唤醒, 直接退出), 唤醒action\-\>thread中断线程

(5) irq\_exit(): 减少当前进程thread\_info的preempt\_count中的HARDIRQ域值, 退出硬件中断上下文; 如果不在中断上下文, 并且本地CPU的\_\_softirq\_pending有pending等待的软中断则处理, 详细见软中断

## 1.5 中断线程执行过程

(1) 如果aciton的flags没设置为IRQTF\_RUNTHREAD, 设置线程为TASK\_INTERRUPTIBLE, 调用schedule(), 换出CPU, 进入睡眠等待

(2) 设置线程为TASK\_RUNNING, 执行action\-\>thread\_fn

# 2 软中断和tasklet

中断线程化属于下半部范畴, 这个机制合并前, 已经有了下半部机制, 例如**软中断**, **tasklet**和**工作队列**.

## 2.1 SoftIRQ软中断

对时间要求最严格和最重要的下半部, 目前驱动只有块设备和网络子系统在使用.

**软中断**是Linux内核中最常见的一种**下半部机制**，适合系统对**性能和实时响应要求很高**的场合，例如**网络子系统**、**块设备**、**高精度定时器**、**RCU**等。

- **软中断**类型是**静态定义**的，Linux内核**不希望**驱动开发者**新增软中断类型**。
- 软中断的**回调函数**在**开中断环境下**执行。
- **同一类型的软中断**可以在**多个CPU**上并行执行。以**TASKLET\_SOFTIRQ**类型的软中断为例，多个CPU可以同时tasklet\_schedule，并且多个CPU也可能同时从中断处理返回，然后同时触发和执行TASKLET\_SOFTIRQ类型的软中断。
- 假如有驱动开发者要新增一个软中断类型，那软中断的处理函数需要考虑同步问题。
- 软中断的**回调函数不能睡眠**。
- 软中断的**执行时间点**是在**中断返回前**，即**退出硬中断上下文**时，首先检查**是否有pending的软中断**，然后才检查**是否需要抢占当前进程**。因此，**软中断上下文总是抢占进程上下文(！！！**)。

10种静态定义的软中断类型, 通过枚举实现, 索引号越小, 软中断优先级越高

描述软中断softirq\_action, 一个**全局软中断描述符数组**, 每种软中断一个

```c
[include/linux/interrupt.h]
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};

[kernel/softirq.c]
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```

每个CPU定义一个软中断状态信息irq\_cpustat\_t

```c
[include/asm-generic/hardirq.h]
typedef struct {
	unsigned int __softirq_pending;
} ____cacheline_aligned irq_cpustat_t;

[kernel/softirq.c]
#ifndef __ARCH_IRQ_STAT
irq_cpustat_t irq_stat[NR_CPUS] ____cacheline_aligned;
EXPORT_SYMBOL(irq_stat);
#endif
```

每个CPU有一个执行软中断的守护进程ksoftirqd(Per\-CPU变量)

注册软中断: 在全局的软中断描述符数组中, 指定相应软中断的action. open\_softirq()

触发软中断: 将本地CPU的软中断状态信息irq\_stat中相应软中断位置为1, 如果不在中断上下文, 唤醒软中断守护进程ksoftirqd, 中断上下文, 退出. raise\_softirq()和raise\_softirq\_irqoff(), 前面主动关闭本地中断, 所以后者允许进程上下文调用

软中断的执行: 

(1) 中断退出阶段执行(irq\_exit()): 在**非中断上下文(!interrupt()**), 以及有**pending**情况下才继续.

\_\_**do\_softirq**():

获取本地CPU的软中断状态irq\_stat, 增加当前进程struct thread\_info中的preempt\_count成员里的SOFTIRQ域的值**SOFTIRQ\_OFFSET(！！！加的值是2的8次方, preempt\_count[8:15]表示软中断, 刚好将bit[8]设为1**), 表明在软中断上下文; 清除本地CPU的所有软中断状态, 因为会一次全部处理; 循环处理软中断, 从索引小的开始, 调用action()函数指针; 如果又有新软中断, 软中断处理时间没超过2毫秒并且没有进程要求调度, 则再处理一次软中断, 否则唤醒ksoftirqd处理 ;退出软中断上下文

中断退出**不能**处于**硬件中断上下文**和**软中断上下文**. 硬中断处理过程一般是关中断的, 中断退出也就退出了硬件中断上下文, 这里肯定会满足; 另一个场景, 本次**中断点**发生在**软中断过程中**, 那中断退出会返回到软中断上下文, 这时候不允许重新调度软中断. 因为软中断在一个CPU上总是串行执行.

(2) ksoftirqd(两个来源\<irq\_exit()\>和主动): 和上面动作类似

## 2.2 tasklet

**tasklet**是基于**软中断**的一种下半部机制, 所以还是运行在软中断上下文。

- tasklet可以**静态定义**，也可以**动态初始化**。
- tasklet是**串行执行**的。一个**tasklet**在**tasklet\_schedule**()时会绑定某个CPU的**tasklet\_vec链表**，它必须要在该CPU上**执行完tasklet的回调函数**才会和该CPU**松绑**。
- **TASKLET\_STATE\_SCHED**和**TASKLET\_STATE\_RUN标志位**巧妙地构成了**串行执行**。
- 同一个tasklet只能同时在一个cpu上执行，但不同的tasklet可以同时在不同的cpu上执行；
- 一旦tasklet\_schedule被调用，内核会保证tasklet一定会在某个cpu上执行一次；
- 如果tasklet\_schedule被调用时，tasklet不是出于正在执行状态，则它只会执行一次；
- 如果tasklet\_schedule被调用时，tasklet已经正在执行，则它会在稍后被调度再次被执行；
- 两个tasklet之间如果有资源冲突，应该要用自旋锁进行同步保护；

tasklet\_struct数据结构:

```c
[include/linux/interrupt.h]
struct tasklet_struct
{
    //多个tasklet串成一个链表
	struct tasklet_struct *next;
	// 该tasklet当前状态.
	unsigned long state;
	// 为0表示tasklet处于激活状态；不为0表示该tasklet被禁止，不允许执行
	atomic_t count;
	// tasklet处理程序，类似软中断中的action函数指针。
	void (*func)(unsigned long);
	// 传递参数给tasklet处理函数
	unsigned long data;
};


enum
{
    // 表示tasklet己经被调度，正准备运行
	TASKLET_STATE_SCHED,	/* Tasklet is scheduled for execution */
	// 表示tasklet正在运行中
	TASKLET_STATE_RUN	/* Tasklet is running (SMP only) */
};
```

每个CPU(实际上是每个logical processor, 即每个cpu thread)维护两个tasklet链表，一个用于普通优先级的tasklet\_vec，另一个用于高优先级的tasklet\_hi\_vec，它们都是Per-CPU变量(！！！)。链表中每个tasklet\_struct代表一个tasklet。

```c
[kernel/softirq.c]
struct tasklet_head {
	struct tasklet_struct *head;
	struct tasklet_struct **tail;
};

static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);
```

其中，tasklet\_vec使用软中断中的TASKLET\_SOFTIRQ类型，它的优先级是6; 而tasklet\_hi\_vec使用的软中断中的HI\_SOFTIRQ, 优先级是0，是所有软中断中优先级最高的。

系统初始化会初始化这两个链表(softirq\_init()), 会注册TASKLET\_SOFTIRQ和HI\_SOFTIRQ这两个软中断(！！！), 回调函数分别是**tasklet\_action**和**tasklet\_hi\_action**(网络驱动用的多)。

```c
[start_kernel()->softirq_init()]
[kernel/softirq.c]
void __init softirq_init(void)
{
	int cpu;

	for_each_possible_cpu(cpu) {
		per_cpu(tasklet_vec, cpu).tail =
			&per_cpu(tasklet_vec, cpu).head;
		per_cpu(tasklet_hi_vec, cpu).tail =
			&per_cpu(tasklet_hi_vec, cpu).head;
	}

	open_softirq(TASKLET_SOFTIRQ, tasklet_action);
	open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

以普通优先级为例.

初始化一个tasklet: 静态(如下)或动态初始化(tasklet\_init())

```c
[include/linux/interrupt.h]
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }
```

调度tasklet的执行: tasklet\_schedule(). 设置tasklet的state为TASKLET\_STATE\_SCHED, 原来已经是TASKLET\_STATE\_SCHED, 说明已经在链表, 退出; 否则将tasklet挂载到tasklet\_vec链表, raise\_softirq\_irqoff()触发软中断

tasklet的执行: 基于软中断机制, 当循环到TASKLET\_SOFTIRQ类型软中断时, 回调函数是tasklet\_action(). 

(1) 获取当前CPU的tasklet链表到一个临时链表, 然后清除当前CPU的, 允许新tasklet进入待处理链表

(2) **遍历临时链表**, tasklet\_trylock判断**当前tasklet**是否已经在其他CPU运行或被禁止

- 没有运行, 也没有禁止, 清除TASKLET\_STATE\_SCHED, 执行回调函数
- 已经在运行或被禁止, 将该tasklet重新添加当当前CPU的待处理tasklet链表, 然后触发TASKLET\_SOFTIRQ序号(6)的软中断, 等下次软中断再执行

**软中断上下文优先级高于进程上下文**，因此**软中断包括tasklet总是抢占进程(！！！**)的运行。当**进程A在运行时发生中断**，在**中断返回**时**先判断本地CPU上有没有pending的软中断**，如果有，那么首先**执行软中断包括tasklet**, 然后**检查是否有高优先级任务需要抢占中断点的进程**，即进程A。如果在执行软中断和tasklet过程时间很长，那么高优先级任务就长时间得不到运行，势必会影响系统的实时性，这也是RT Linux社区里有专家一直**要求用workqueue机制来替代tasklet机制**的原因。

![config](./images/10.png)

目前Linux内核中有大量的驱动程序使用tasklet机制来实现下半部操作，任何一个tasklet回调函数执行时间过长，都会影响系统实时性，可以预见在不久的将来**tasklet机制**有可能会被Linux内核社区**舍弃**。

## 2.3 local\_bh\_disable/local\_bh\_enable下半部临界区

内核中提供的关闭软中断的锁机制，它们组成的临界区**禁止本地CPU**在**中断返回前(！！！**)夕**执行软中断**，这个临界区简称BH临界区(bottom half critical region).

local\_bh\_disable: **关闭软中断**. 将当前进程preempt\_count加上SOFTIRQ\_DISABLE\_OFFSET(该值为512, 2的9次方, 参考preempt\_count结构, bit[8:15]表示软中断, 该域还表示软中断嵌套深度, 所以9次方, bit[9]是1, 在软中断这里是2, 两层嵌套), 表明进入了**软中断上下文**, 这样中断返回前irq\_exit()不能调用执行pending状态的软中断

local\_bh\_enable: 打开软中断. preempt\_count先减去(SOFTIRQ\_DISABLE\_OFFSET \- 1), 表明**退出了软中断上下文(bit[8:15]已经是0了**), 剩1表示**关闭本地CPU抢占(参见preempt\_count组成**), 因为不希望被其他高优先级任务抢占当前; 在**非中断上下文**执行**软中断处理**, 走上面软中断流程

在**进程上下文调用建立临界区**, 此时来了**外部中断**后, 当*中断返回*时, 发现处于**软中断上下文**, 那么就**不执行, 延迟**了.

## 2.4 中断上下文

**中断上下文**包括**硬中断上下文**（hardirq context)和**软中断上下文**（softirq context)。

- **硬件中断上下文**表示**硬件中断处理过程**。
- **软中断上下文**包括**三部分**
    - 一是在**下半部执行的软中断处理包括tasklet**，调用过程是**irq\_exit()\->invoke\_softirq**();
    - 二是**ksoftirqd内核线程执行的软中断**，例如系统使能了**强制中断线程化**force\_irqthreads (见invoke\_softirq()函数)，还有一种情况是**软中断执行时间太长**，在\_do\_softirq()中**唤醒ksoftirqd内核线程**；
    - 三是**进程上下文(！！！**)中调用**local\_bh\_enable**()时也会去**执行软中断处理**，调用过程是**local\_bh\_enable()-〉do\_softirq**()。

软中断上下文中前者**调用在中断下半部**中，属于传统意义上的**中断上下文**，而**后两者(！！！)调用在进程上下文中**，但是Linux内核统一把它们归纳到软中断上下文范畴里。

preempt\_count成员在第3.1节中(进程管理)介绍过，如图5.6所示。

![config](./images/1.png)

**中断上下文(！！！**)包括**硬件中断处理过程**、**关BH临界区**、**软中断处理过程(！！！**)和**NMI中断**处理过程。在内核代码中经常需要判断当前状态是否处于进程上下文中，也就是希望确保当前不在任何中断上下文中，这种情况很常见，因为代码需要做一些睡眠之类的事情。**in\_interrupt**()宏返回false,则此时内核处于**进程上下文**中，否则处于**中断上下文**中。

Linux内核中有几个宏来描述和判断这些情况：

```c
[include/linux/preempt_mask.h]

#define hardirq_count()	(preempt_count() & HARDIRQ_MASK)
#define softirq_count()	(preempt_count() & SOFTIRQ_MASK)
#define irq_count()	(preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK \
				 | NMI_MASK))

#define in_irq()		(hardirq_count())
#define in_softirq()		(softirq_count())
#define in_interrupt()		(irq_count())
#define in_serving_softirq()	(softirq_count() & SOFTIRQ_OFFSET)
```

- in\_irq()判断当前是否在**硬件中断上下文**中;
- in\_softirq()判断当前是否在**软中断上下文**中或者**处于关BH的临界区(！！！**)里；
- in\_serving\_softirq()判断当前是否正在**软中断处理(！！！**)中，包括前文提到的**三种情况**。
- in\_interrupt()则包括所有的**硬件中断上下文**、**软中断上下文**和**关BH临界区**。

