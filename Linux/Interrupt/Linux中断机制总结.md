
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Linux整体机制](#1-linux整体机制)
	* [1.1 中断初始化](#11-中断初始化)
		* [1.1.1 中断处理接口interrupt数组](#111-中断处理接口interrupt数组)
	* [1.2 硬件中断号和软件中断号映射](#12-硬件中断号和软件中断号映射)
	* [1.3 注册中断](#13-注册中断)
	* [1.4 底层中断处理](#14-底层中断处理)
	* [1.5 高层中断处理](#15-高层中断处理)
	* [1.6 中断线程执行过程](#16-中断线程执行过程)
* [2 软中断和tasklet](#2-软中断和tasklet)
	* [2.1 SoftIRQ软中断](#21-softirq软中断)
	* [2.2 tasklet](#22-tasklet)
	* [2.3 local\_bh\_disable/local\_bh\_enable下半部临界区](#23-local_bh_disablelocal_bh_enable下半部临界区)
	* [2.4 中断上下文](#24-中断上下文)
* [3 workqueue](#3-workqueue)
	* [3.1 背景和原理](#31-背景和原理)
	* [3.2 数据结构](#32-数据结构)

<!-- /code_chunk_output -->

# 1 Linux整体机制

中断在那个CPU上执行，取决于在**那个CPU上申请了vector**并**配置了对应的中断控制器(比如local APIC**)。如果想要**改变一个中断的执行CPU**，必须**重新申请vector并配置中断控制器**。一般通过**echo xxx > /proc/irq/xxx/affinity**来完成调整，同时irq\_balance一类软件可以用于完成中断的均衡。

当外设触发一次中断后，一个大概的处理过程是：

1、具体CPU architecture相关的模块会进行现场保护

2、通过IDT中的中断描述符，调用common_interrupt； 

3、通过common\_interrupt，调用do\_IRQ，完成vector到irq\_desc的转换(根据硬件的信息获取HW interrupt ID，并且通过irq domain模块翻译成IRQ number)，进入Generic interrupt layer(调用处理函数generic\_handle\_irq\_desc)； 

4、调用在**中断初始化的时候**，按照**中断特性**(level触发，edge触发等、simple等)初始化的irq\_desc:: handle\_irq，执行不同的通用处理接口，比如handle\_simple\_irq; 调用该IRQ number对应的high level irq event handler，在这个high level的handler中，会通过和interupt controller交互，进行中断处理的flow control（处理中断的嵌套、抢占等），当然最终会遍历该中断描述符的IRQ action list，调用外设的specific handler来处理该中断

5、这些通用处理接口会调用中断初始化的时候注册的外部中断处理函数；完成EOI等硬件相关操作；并完成中断处理的相关控制。

6、具体CPU architecture相关的模块会进行现场恢复。

## 1.1 中断初始化

对X86 CPU，Linux内核使用全局idt\_table来表达当前的IDT，该变量定义在traps.c

```c
[arch/x86/kernel/traps.c]
gate_desc idt_table[NR_VECTORS] __page_aligned_bss;
```

对中断相关的初始化，内核主要有以下工作： 

1、 设置used\_vectors，确保外设不能分配到X86保留使用的vector(**预留的vector范围为[0,31**]，另外还有其他通过apic\_intr_init等接口预留的系统使用的vector); 

2、 设置X86CPU保留使用的vector对应的IDT entry;这些entry使用特定的中断处理接口； 

3、 设置外设 (包括ISA中断)使用的中断处理接口，这些中断处理接口都一样。 

4、 设置ISA IRQ使用的irq_desc； 

5、 把IDT的首地址加载到CPU的IDTR(Interrupt Descriptor Table Register); 

6、 初始化中断控制器(下一章描述) 

以上工作主要在以下函数中完成：

```c
start_kernel
    trap_init 
        使用set_intr_gate等接口初始化保留vector
        used_vectors中[0,0x1f]对应的vector被置1，表示已经预留不能再使用
        cpu_init
            load_idt((const struct desc_ptr *)&idt_descr) 把&idt_descr的地址加载到idtr
                native_load_idt()
    init_IRQ
        初始化0号CPU的vector_irq：其vector[0x30-0x3f]和irq号[0-15](ISA中断)对应
        x86_init.irqs.intr_init(native_init_IRQ)
            x86_init.irqs.pre_vector_init(init_ISA_irqs)
                init_ISA_irqs:初始化ISA，设置irq号[0,15] (ISA中断)的irq_desc
            apic_intr_init 设置APIC相关中断(使用的alloc_intr_gate会设置used_vectors)
            使用interrupt数组初始化设置外设中断idt entry
```

这个过程会完成每个中断vector对应的idt entry的初始化，系统把这些中断vector分成以下几种： 

1、**X86保留vector**，这些**vector包括[0,0x1f**]和**APIC等系统部件占用的vector**,对这些vector，会**记录在bitmap used\_vectors中**，确保**不会被外设分配使用**；同时这些vector都使用**各自的中断处理接口(！！！**)，其中断处理过程相对简单(没有generic interrupt layer的参与，CPU直接调用到各自的ISR)。 

2、**ISA irqs**，对这些中断，在初始化过程中已经完成了**irq\_desc**、**vector\_irq**、以及**IDT中对应entry的分配和设置**，同时可以发现**ISA中断**，在初始化的时候都被设置为**运行在0号CPU**。 0x30\-0x3f是ISA的

3、**其它外设的中断**，对这些中断，在初始化过程中**仅设置了对应的IDT**，和**ISA中断一样**，其**中断处理接口都来自interrupt数组**。

### 1.1.1 中断处理接口interrupt数组 

interrupt数组是内核中外设中断对应的IDT entry，其在arch/x86/entry/entry\_64.S中定义，定义如下：

```assembly
[arch/x86/include/asm/irq_vectors.h]
#define FIRST_EXTERNAL_VECTOR		0x20
#define LOCAL_TIMER_VECTOR		0xef

#define NR_VECTORS			 256

#ifdef CONFIG_X86_LOCAL_APIC
#define FIRST_SYSTEM_VECTOR		LOCAL_TIMER_VECTOR
#else
#define FIRST_SYSTEM_VECTOR		NR_VECTORS
#endif

[arch/x86/entry/entry_64.S]
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
    /* 代码段内容*/
	pushq	$(~vector+0x80)			/* Note: always in signed byte range */
    vector=vector+1
	jmp	common_interrupt
	.align	8
    .endr
END(irq_entries_start)
```

这段汇编的效果是：在**代码段**，生成了一个**符号irq\_entries\_start**，该符号对应的内容是一组可执行代码.

X86 CPU在准备好了中断执行环境后，会调用中断描述符定义的中断处理入口；对于**用户自定义中断**，中断处理入口都是(对系统预留的，就直接执行定义的接口了)common\_interrupt

## 1.2 硬件中断号和软件中断号映射

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

## 1.3 注册中断

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

## 1.4 底层中断处理

具体看体系结构

## 1.5 高层中断处理

common\_interrupt在entry\_64.S中定义，其中关键步骤为：调用**do\_IRQ**，完成后会根据**环境**判断**是否需要执行调度**，最后执行**iretq指令**完成中断处理，iret指令的重要功能就是**恢复中断函数前的EFLAGS(执行中断入口前被入栈保存，并清零IF位关中断**)，并恢复执行被中断的程序(这里不一定会恢复到之前的执行环境，可能执行软中断处理，或者执行调度)。

do_IRQ的基本处理过程如下，其负责中断执行环境建立、vector到irq的转换等, 调用内核代码继续处理

(1) irq\_enter(): 增加当前进程thread\_info的preempt\_count中的HARDIRQ域值, 表明进入硬件中断上下文

(2) irq\_find\_mapping()通过硬件中断号查找IRQ中断号(irq domain模块中R树或数组)

(3) 获得中断描述符, 调用desc\-\>**handle\_irq**(), 即high level event handler, 不同触发类型不同high level event handler. 基本操作都是中断响应相关, 然后遍历中断描述符的action链表, 依次执行每个action的primary handler, 一般而言, primary handler编写最开始要通过dev\_id确认是否是自己的设备产生了中断

(4) primary handler如果返回IRQ\_WAKE\_THREAD(默认primary就是直接返回这个),设置aciton的flags为IRQTF\_RUNTHREAD(已经置位说明已经唤醒, 直接退出), 唤醒action\-\>thread中断线程

(5) irq\_exit(): 减少当前进程thread\_info的preempt\_count中的HARDIRQ域值, 退出硬件中断上下文; 如果不在中断上下文, 并且本地CPU的\_\_softirq\_pending有pending等待的软中断则处理, 详细见软中断

## 1.6 中断线程执行过程

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

![config](./images/6.png)

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

# 3 workqueue

## 3.1 背景和原理

工作队列的基本原理是把work(需要推迟执行的函数）交由一个**内核线程**来执行，它总是在**进程上下文**中执行。工作队列的优点是利用进程上下文来执行中断下半部操作，因此工作队列允许**重新调度**和**睡眠**，是**异步执行的进程上下文**，另外它还能解决**软中断**和**tasklet**执行**时间过长**导致系统**实时性下降**等问题。

早起workqueue比较简单, 由多线程（Multi threaded，每个CPU默认一个工作线程）和单线程（Single threaded, 用户可以自行创建工作线程）组成. 容易导致①内核线程数量太多 ②并发性差(工作线程和CPU是绑定的) ③死锁(同一个队列上的数据有依赖容易死锁)

concurrency\-managed workqueues(CMWQ): BOUND类型(Per\-CPU, 每个CPU一个)和UNBOUND类型, **每种**都有**两个工作线程池(worker\-pool**): 普通优先级的工作(work)使用和高优先级的工作(work)使用. 工作线程池(worker\-pool)中线程数量是动态管理的. 工作线程睡眠时, 检查是否需要唤醒更多工作线程, 有需要则唤醒同一个工作线程池中idle状态的工作线程.

## 3.2 数据结构

工作任务struct work\_struct

```c
[include/linux/workqueue.h]
struct work_struct {
    //低比特位是标志位, 剩余存放上一次运行的worker_pool的ID或pool_workqueue的指针(由WORK_STRUCT_PWQ标志位来决定)
	atomic_long_t data;
	// 把work挂到其他队列上
	struct list_head entry;
	// 工作任务处理函数
	work_func_t func;
};
```

工作线程struct worker

```c
[kernel/workqueue_internal.h]
struct worker {
	/* on idle list while idle, on busy hash table while busy */
	union {
		struct list_head	entry;	/* L: while idle */
		struct hlist_node	hentry;	/* L: while busy */
	};
    // 正在被处理的work
	struct work_struct	    *current_work;	/* L: work being processed */
	// 正在执行的work回调函数
	work_func_t		        current_func;	/* L: current_work's fn */
	// 当前work所属的pool_workqueue
	struct pool_workqueue	*current_pwq;   /* L: current_work's pwq */
	// 所有被调度并正准备执行的work都挂入该链表
	struct list_head	    scheduled;	    /* L: scheduled works */
    // 挂入到worker_pool->workers链表
    struct list_head	    node;		    /* A: anchored at pool->workers */
						                    /* A: runs through worker->node */
	// 工作线程的task_struct
	struct task_struct	    *task;		    /* I: worker task */
	// 该工作线程所属的worker_pool
	struct worker_pool	    *pool;		    /* I: the associated pool */
	// 该工作线程的ID号
	int			            id;		        /* I: worker id */
    ...
};
```

工作线程池struct worker\_pool

```c
[kernel/workqueue.c]
struct worker_pool {
    // 保护worker-pool的自旋锁
	spinlock_t		lock;		/* the pool lock */
	// BOUND类型的workqueue，cpu表示绑定的CPU ID; UNBOUND类型，该值为-1
	int			    cpu;		/* I: the associated cpu */
	// UNBOUND类型的workqueue，表示该worker-pool所属内存节点的ID编号
	int			    node;		/* I: the associated node ID */
	// ID号
	int			    id;		    /* I: pool ID */
	unsigned int	flags;		/* X: flags */
    // pending状态的work会挂入该链表
	struct list_head	worklist;	/* L: list of pending works */
	// 工作线程的数量
	int			        nr_workers;	/* L: total number of workers */
	// idle状态的工作线程的数量
	int			        nr_idle;	/* L: currently idle ones */
    // idle状态的工作线程挂入该链表
	struct list_head	idle_list;	/* X: list of idle workers */
	// 被管理的工作线程会挂入该链表
	struct list_head	workers;	/* A: attached workers */
	// 工作线程的属性
	struct workqueue_attrs	*attrs;		/* I: worker attributes */
	// 正在运行中的worker数量
	atomic_t		nr_running ____cacheline_aligned_in_smp;
	// rcu锁
	struct rcu_head		rcu;
	...
} ____cacheline_aligned_in_smp;
```

连接**workqueue(工作队列**)和**worker\_pool(工作线程池**)的桥梁**struct pool\_workqueue**

```c
[kernel/workqueue.c]
struct pool_workqueue {
    // worker_pool指针
	struct worker_pool	    *pool;		/* I: the associated pool */
	// 工作队列
	struct workqueue_struct *wq;		/* I: the owning workqueue */
	// 活跃的work数量
	int			            nr_active;	/* L: nr of active works */
	// 活跃的work最大数量
	int			            max_active;	/* L: max active works */
	// 被延迟执行的works挂入该链表
	struct list_head	    delayed_works;	/* L: delayed works */
	struct rcu_head		    rcu;
	...
} __aligned(1 << WORK_STRUCT_FLAG_BITS);
```

工作队列struct workqueue\_struct

```c
[kernel/workqueue.c]
struct workqueue_struct {
    // 所有的pool-workqueue数据结构都挂入链表
	struct list_head	pwqs;		/* WR: all pwqs of this wq */
	// 链表节点。当前workqueue挂入全局的链表workqueues
	struct list_head	list;		/* PL: list of all workqueues */
    // 所有rescue状态下的pool-workqueue数据结构挂入该链表
	struct list_head	maydays;	/* MD: pwqs requesting rescue */
	// rescue内核线程. 创建workqueue时设置WQ_MEM_RECLAIM, 那么内存紧张而创建新的工作线程失败会被该线程接管
	struct worker		*rescuer;	/* I: rescue worker */
    // UNBOUND类型属性
	struct workqueue_attrs	*unbound_attrs;	/* WQ: only for unbound wqs */
	// UNBOUND类型的pool_workqueue
	struct pool_workqueue	*dfl_pwq;	/* WQ: only for unbound wqs */
    // workqueue的名字
	char			name[WQ_NAME_LEN]; /* I: workqueue name */

	/* hot fields used during command issue, aligned to cacheline */
	unsigned int		flags ____cacheline_aligned; /* WQ: WQ_* flags */
	//Per-CPU类型的pool_workqueue
	struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
	...
};
```

关系图:

**一个work挂入workqueue**中，最终还要**通过worker\-pool**中的**工作线程来处理其回调函数**，worker-pool是**系统共享的(！！！**)，因此**workqueue**需要查找到一个**合适的worker\-pool**，然后从worker\-pool中分派一个**合适的工作线程**，pool\_workqueue数据结构在其中起到**桥梁**作用。这有些类似IT类公司的人力资源池的概念，具体关系如图5.7所示。

![config](./images/2.png)

![config](./images/3.gif)

![config](./images/4.png)

- **work\_struct**结构体代表的是**一个任务**，它指向一个待异步执行的函数，不管驱动还是子系统什么时候要执行这个函数，都必须把**work**加入到一个**workqueue**。

- **worker**结构体代表一个**工作者线程（worker thread**），它主要**一个接一个的执行挂入到队列中的work**，如果没有work了，那么工作者线程就挂起，这些工作者线程被worker\-pool管理。

对于**驱动和子系统**的开发人员来说，接触到的**只有work**，而背后的处理机制是管理worker\-pool和处理挂入的work。

- **worker\_pool**结构体用来**管理worker**，对于**每一种worker pool**都分**两种情况**：一种是处理**普通work**，另一种是处理**高优先级的work**。

- **workqueue\_struct**结构体代表的是**工作队列**，工作队列分**unbound workqueue**和**bound workqueue**。bound workqueue就是**绑定到cpu**上的，**挂入到此队列中的work**只会在**相对应的cpu**上运行。**unbound workqueue不绑定到特定的cpu**，而且**后台线程池的数量也是动态**的，具体**workqueue关联到哪个worker pool**，这是由**workqueue\_attrs决定**的。

系统初始化阶段(init\_workqueue()): 为**所有CPU(包括离线**的)创建**两个工作线程池worker\_pool**(普通优先级和高优先级); 为每个在线CPU的每个工作线程池(每个CPU有两个)创建一个工作线程(create\_worker); 创建UNBOUND类型和ordered类型的workqueue属性, 分别两个, 对应普通优先级和高优先级, 供后续使用; alloc\_workqueue()创建几个默认的workqueue

- **普通优先级BOUND类型**的**工作队列system\_wq**, 名称为“**events**”，可以理解为**默认工作队列**。
- **高优先级BOUND类型**的工作队列**system\_highpri\_wq** ，名称为“**events\_highpri**”。
- **UNBOUND类型**的工作队列**system\_unbound\_wq**，名称为“**system\_unbound\_wq**”。
- **Freezable类型**的工作队列**system\_freezable\_wq**，名称为“**events\_freezable**”。
- **省电类型**的工作队列**system\_freezable\_wq**，名称为 “**events\_power\_efficient**”。

创建工作线程worker(参数是worker\_pool): 获取一个ID; 工作线程池对应的内存节点分配一个worker; 在工作线程池对应的node创建一个内核线程, 名字("**kworker/u \+ CPU\_ID \+ : \+ worker\_idH**", 高优先级的才有H, UNBOUND类型的才有u); 设置线程(worker->task->flags)的PF\_NO\_SETAFFINITY标志位(防止修改CPU亲和性); 工作线程池没有绑定到CPU上, 那么设置worker标志位不绑定CPU; 将worker加到工作线程池的workers链表; 使worker进入idle状态; 唤醒worker的内核线程; 返回该worker

**创建工作队列workqueue**: API很多, 3个参数name, flags和max\_active.

(1) **分配一个workqueue\_struct**并初始化, 对于UNBOUND类型, 创建一个UNBOUND类型的workqueue属性

(2) **分配pool\_workqueue**并初始化alloc\_and\_link\_pwqs()
    
BOUND类型的workqueue: 

- 给**每个CPU**分配一个Per\-CPU的**pool\_workqueue**, 
- 遍历每个CPU, 通过**这个pwq**将系统静态定义的Per\-CPU类型的**高优先级的worker\_pool**(也就是init\_workqueues()初始化的)和**workqueue**连接起来, 并将这个pool\_workqueue添加到传入的workqueue\-\>pwqs链表.

UNBOUND类型和ORDERED类型的workqueue: 都是调用apply\_workqueue\_attrs实现, 不同在于传入的属性一个是ordered\_wq\_attrs[highpri], 一个是unbound\_std\_wq\_attrs[highpri], 这两个不同在于属性里面的no\_numa在ordered中是true, 这两个属性系统初始化阶段完成的. 

- 通过系统**全局哈希表unbound\_pool\_hash**(管理所有UNBOUND类型的work\_pool)根据属性查找**worker\_pool**, 找到将其引用计数加1, 并返回, 没有的话重新分配并初始化一个(创建pool, 为pool创建一个**工作线程worker**<会唤醒线程>), 将新pool加入哈希表
- 分配一个pool\_workqueue
- 初始化该pwq, 将worker\_pool和workqueue\_struct连接起来, 为pool\_workqueue初始化一个**工作work**(通过INIT\_WORK()), 回调函数是pwq\_unbound\_release\_workfn(), 该work执行: 从work中找到相应的pwq, 该work只对UNBOUND类型的workqueue有效, 如果work\-\>pwq\-\>wq\-\>pwqs(所有pool\_workqueue都在这个链表)中当前pool\_workqueue是最后一个, 释放pool\_workqueue相关结构

**初始化一个work**: 宏INIT\_WORK(work, func)

**调度一个work**: schedule\_work(work\_struct), 将work挂入系统默认的BOUND类型的workqueue工作队列system\_wq, queue\_work(workqueue, work)

(1) 关中断

(2) 设置work标志位WORK\_STRUCT\_PENDING\_BIT, 已经有说明正在pending, 已经在队列中, 不用重复添加

(3) 找一个合适的pool\_workqueue. 优先本地CPU或本地CPU的node节点对应的pwq, 如果该work**上次执行的worker\_pool**和刚选择的pwq\-\>pool不等, 并且**该work**正在其**上次执行的工作线程池中运行**，而且运行**这个work的worker对应的pwq**对应的workqueue等于调度**传入的workqueue**(worker\-\>current\_pwq\-\>wq == wq), 则优先选择这个正在运行的worker\-\>current\_pwq. 利用其**缓存热度**.

(4) 判断当前pwq活跃的work数量, 少于最高限值, 加入pwq\-\>pool\-\>worklist(pool的pending链表), 否则加入pwq\-\>delayed\_works(pwq的被延迟执行的works链表)

(5) 当前pwq\-\>pool工作线程池存在pending状态的work并且pool中正运行的worker数量为0的话, 找到pool中第一个idle的worker并唤醒worker\-\>task

(6) 开中断

工作线程处理函数worker\_thread():

```c
worker_thread()
{
recheck:
    if(不需要更多的工作线程?)
        goto 睡眠;
        
    if(需要创建更多的工作线程? && 创建线程)
        goto recheck;
        
    do{
        处理工作;
    }(还有工作待完成 && 活跃的工作线程 <= 1) // 这儿就是keep_working(pool)
    
睡眠:
    schedule();
}
```

- 动态地创建和管理一个工作线程池中的工作线程。假如发现有PENDING的work且当前工作池中没有正在运行的工作线程（worker\_pool\-\>nr\_running = 0)，那就唤醒idle状态的线程，否则就动态创建一个工作线程。
- 如果发现一个work己经在同一个工作池的另外一个工作线程执行了，那就不处理该work。
- 动态管理活跃工作线程数量，见keep\_working()函数。如果pool\-\>worklist中还有工作需要处理且工作线程池中活跃的线程小于等于1，那么保持当前工作线程继续工作，此功能可以防止工作线程泛滥。也就是限定活跃的工作线程数量小于等于1.

和调度器交互:

CMWQ机制会动态地调整一个线程池中工作线程的执行情况，不会因为某一个work回调函数执行了阻塞操作而影响到整个线程池中其他work的执行。

某个work的回调函数func()中执行了睡眠操作(设置当前进程state为TASK\_INTERRUPTIBLE, 然后执行schedule()切换), 在**schedule**()中, 判断**进程的flags是否有PF\_WQ\_WORKER(属于worker线程**), 有的话:

(1) 将**当前worker的worker\_pool中nr\_running引用计数减1**, 如果为0则说明**当前线程池没有活跃的工作线程**, 而**当前线程池的等待队列worklist**有work, 那么从**pool\-\>idle\_list**链表**拿一个idle的工作线程**

(2) 唤醒该工作线程, 增加worker\_pool中nr\_running引用计数

- **工作线程进入执行**时会增加nr\_running 计数，见**worker\_thread**()\-〉worker\_clr\_flags()函数。
- 工作线程**退出执行**时会减少nr\_running 计数，见worker\_thread()\-〉worker\_set\_flags()函数。
- **工作线程进入睡眠**时会减少nr\_running计数，见\_\_**schedule**()函数。
- 工作线程**被唤醒时**会增加nr\_running计数，见ttwu\_activate()函数。

在驱动开发中使用workqueue是比较简单的，特别是**使用系统默认的工作队列system\_wq**, 步骤如下。

- 使用**INIT\_WORK**()宏声明一个work和该work的回调函数。
- **调度一个work**: **schedule\_work**()。
- **取消一个work**: **cancel\_work\_sync**()
 
此外，有的驱动程序还**自己创建一个workqueue**，特别是**网络子系统**、**块设备子系统**等。

- 使用**alloc\_workqueue**()创建**新的workqueue**。
- 使用**INIT\_WORK**()宏声明一个**work**和**该work的回调函数**。
- 在**新workqueue**上**调度一个work**: **queue\_work**()
- **flush workqueue**上**所有work**: flush\_workqueue()

Linux内核还提供一个**workqueue机制**和**timer机制**结合的**延时机制delayed\_work**

要**理解CMWQ机制**，首先要明白旧版本的workqueue机制遇到了哪些问题，其次要清楚CMWQ机制中几个重要数据结构的关系. **CMWQ机制**把**workqueue**划分为**BOUND类型**和**UNBOUND类型**。

如图5.8所示是**BOUND类型workqueue机制**的架构图，对于**BOUND类型的workqueue**归纳如下。

- **每个**新建的workqueue，都有一个struct **workqueue\_struct**数据结构来描述。
- 对于**每个新建的(！！！)workqueue**，**每个CPU**有**一个pool\_workqueue(！！！**)数据结构来**连接workqueue**和**worker\_pool**.
- **每个CPU只有两个worker\_pool**数据结构来描述**工作池**，一个用于**普通优先级工作线程**，另一个用于**高优先级工作线程**。
- **worker\_pool**中可以有**多个工作线程**，动态管理工作线程。
- **worker\_pool**和**workqueue**是**1:N(！！！**)的关系，即**一个worker\_pool**可以对应**多个workqueue**.
- pool\_workqueue是worker\_pool和workqueue之间的桥梁枢纽。
- **worker\_pool(！！！**)和**worker工作线程(！！！**)也是**1:N(！！！**)的关系。

![config](./images/5.png)

**BOUND类型的work**是在**哪个CPU**上运行的呢？有几个API接口可以把**一个work**添加到**workqueue**上运行，其中**schedule\_work**()函数**倾向于使用本地CPU**，这样有利于利用**CPU的局部性原理**提高效率，而**queue\_work\_on**()函数可以**指定CPU**的。

对于**UNBOUND类型的workqueue**来说，其**工作线程没有绑定到某个固定的CPU**上。对于**UMA**机器，它可以在**全系统的CPU**内运行；对于**NUMA**机器，**每一个node**节点**创建一个worker\_pool**。

在**驱动开发**中，**UNBOUND类型**的**workqueue不太常用**，举一个典型的例子，Linux内核中有一个**优化启动时间(boot time**)的新接口Asynchronous functioncalls，实现是在kernel/asyn.c文件中。对于一些**不依赖硬件时序**且不需要串行执行的初始化部分，可以采用这个接口，现在电源管理子系统中有一个选项可以把一部分外设在suspend/resume过程中的操作用异步的方式来实现，从而优化其suspend/resume时间，详见kemel/power/main.c 中关于“pm\_async\_enabled” 的实现。

对于**长时间占用CPU资源**的一些负载(标记**WQ\_CPU\_INTENSIVE**), Linux内核倾向于**使用UNBOUND类型的workqueue**, 这样可以利用**系统进程调度器**来优化选择在哪个CPU上运行，例如**drivers/md/raid5.c**驱动。

如下动态管理技术值得读者仔细品味。

- 动态管理**工作线程数量**，包括**动态创建工作线程**和**动态管理活跃工作线程**等。
- 动态**唤醒工作线程**。