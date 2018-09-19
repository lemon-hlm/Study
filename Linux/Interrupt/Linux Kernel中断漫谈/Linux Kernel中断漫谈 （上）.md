- 1 概述
- 2 中断
    - 2.1 x86中断子系统
        - 2.1.1 Intel 8259A芯片
        - 2.1.2 APIC
    - 2.2 ARM中断子系统
    - 2.3 SMP IRQ affinity

# 1 概述

**中断**是穿透**Hardware**和**Software**之间的一种**媒介**。这边是磁盘，鼠标，键盘等等实体；而在另一端是操作系统，驱动或者某个简单的嵌入式程序。当然，中断并不是唯一的方式，具体而言，有**三种经典的方式**：

**轮询**: **周期性的查询外部设备**是否可用。缺点是**浪费是CPU资源**。但是在CPU资源富裕（非计算密集型）或者careful design的情况下，能提供比中断更佳好的性能。比如network中的NAPI（new API）就会在收集到足够多的报文以后，一次性中断CPU，防止每次报文reach都需要中断CPU

**中断**：通过设备的**中断线路**来提醒CPU，sth happends。中断的实现需要**硬件级别的支持**，包括设备，中断芯片和CPU。CPU会在每个cycle的末尾作一次中断的check（硬件级别的check，任何cycle都会有，所以没有性能影响）

**DMA**：**单独的DMA芯片**去负责**数据传输**。对于磁盘之间大数据量的交换非常有效果。DMA会**首先去申请总线**，if成功，那么会使用总线，用完归还。DMA只会在**所有数据处理完毕**后，通过**中断通知CPU**。

早期嵌入式设备的端口可能支持轮询，而更加**新的端口**可以使用**DMA的模式**在完成大块数据的传输工作。

# 2 中断

对于Linux而言，中断进一步准确的分为：

- 异常（excepption）：由CPU内部引起，也叫同步中断，不能被CPU屏蔽，细分分为：
  
    - Fault（出错）：可恢复，恢复后回到出发Fault的那条指令继续执行，比如 Page Fault，在page被load进内存后继续执行刚才执行的命令
    - Trap（陷入）：可恢复，恢复后执行触发Trap指令的后一条指令。比如除0错误，或者Invalid Memory Address Reference。也称为硬陷入（Hardware Trap），对比软陷入（Software Trap）
    - Abort（中止）：无法恢复，比如发生了硬件问题。

- 中断：也叫异步中断，由外部设备引起，细分为可屏蔽中断（INTR）和非可屏蔽中断（NMI）

以上是真正意义上的中断，有各自的IRQ向量和处理函数，也称为硬中断（Hardware Interrupt）。对于X86有两根中断的引脚，一根非屏蔽中断线（包括异常），一根可中断线（接到中断处理芯片）。其中非屏蔽中断线多半是intel内部保留；可中断线一般给外设使用。

对于Liunx而言，它提供了一些的**syscall（INT 0x80**），通过**类似中断的机制陷入到Kernel Space**完成需要的工作（比如读取文件），一般称之为**软中断（Software Interrupt**）。

基于Intel芯片的Linux对于中断按照：异常 -> 非可屏蔽中断 -> 可屏蔽中断 -> 软中断的顺序进行中断向量的分布：

- 0\-15 异常，intel内部保留，不可屏蔽
- 16\-31 非屏蔽中断，intel内部保留，不可屏蔽
- 32\-47 可屏蔽中断，可屏蔽
- 47\-255 软中断，linux系统本身使用128（0x80）来作为系统调用软中断，可屏蔽

## 2.1 x86中断子系统

X86通过（Intel 8259A）两级芯片或者新的APIC（Advanced Programmable Interrupt Controller ）来注册设备。在real mode支持15个中断IRQ0 - IRQ15 (IRQ2被占用)。其中部分的中断号已经被固定（比如给键盘），无法改变，其他的映射到系统中的各个设备（端口）上。

Intel 8259A芯片只适用于**uniprocessor的系统**，对于SMP系统，新的APIC芯片被引入。在APIC中，原有的IRQ0 \- IRQ15分配**不一定保留**。

Intel 8259A的IRQ0 - IRQ15一个典型分配如下：

Master PIC

- IRQ 0 – system timer (cannot be changed)
- IRQ 1 – keyboard controller (cannot be changed)
- IRQ 2 – cascaded signals from IRQs 8–15 (any devices configured to use IRQ 2 will actually be using IRQ 9)
- IRQ 3 – serial port controller for serial port 2 (shared with serial port 4, if present)
- IRQ 4 – serial port controller for serial port 1 (shared with serial port 3, if present)
- IRQ 5 – parallel port 2 and 3 or sound card
- IRQ 6 – floppy disk controller
- IRQ 7 – parallel port 1. It is used for printers or for any parallel port if a printer is not present. It can also be potentially be shared with a secondary sound card with careful management of the port.

Slave PIC

- IRQ 8 – real-time clock (RTC)
- IRQ 9 – Advanced Configuration and Power Interface (ACPI) system control interrupt on Intel chipsets.[1] Other chipset manufacturers might use another interrupt for this purpose, or make it available for the use of peripherals (any devices configured to use IRQ 2 will actually be using IRQ 9)
- IRQ 10 – The Interrupt is left open for the use of peripherals (open interrupt/available, SCSI or NIC)
- IRQ 11 – The Interrupt is left open for the use of peripherals (open interrupt/available, SCSI or NIC)
- IRQ 12 – mouse on PS/2 connector
- IRQ 13 – CPU co-processor or integrated floating point unit or inter-processor interrupt (use depends on OS)
- IRQ 14 – primary ATA channel (ATA interface usually serves hard disk drives and CD drives)
- IRQ 15 – secondary ATA channel

其中**IRQ0（时钟中断**）和**IRQ1（键盘中断**）这两个中断即使在**APIC中依然被保留**。

```
[root@centos7 ~]# cat /proc/interrupts 
           CPU0       CPU1       CPU2       CPU3       
  0:         45          0          0          0   IO-APIC-edge      timer
  1:          1          1          1          0   IO-APIC-edge      i8042
  8:          1          0          0          0   IO-APIC-edge      rtc0
  9:          4          0          0          0   IO-APIC-fasteoi   acpi
 12:          1          0          2          1   IO-APIC-edge      i8042
 16:         10          1         12          2   IO-APIC-fasteoi   ehci_hcd:usb1
 23:         13          1         14          1   IO-APIC-fasteoi   ehci_hcd:usb2
 24:          0          0          0          0   PCI-MSI-edge      PCIe PME, pciehp
 25:          0          0          0          0   PCI-MSI-edge      PCIe PME
 26:        117         14        290          5   PCI-MSI-edge      xhci_hcd
 27:      12853          5         57          0   PCI-MSI-edge      p4p1
 28:       1085        488       3997       2156   PCI-MSI-edge      0000:00:1f.2
 29:          3          2         24         10   PCI-MSI-edge      i915
 30:         38          5        420          0   PCI-MSI-edge      snd_hda_intel
 31:          0          0         10          1   PCI-MSI-edge      mei_me
 32:         26          9         23          0   PCI-MSI-edge      snd_hda_intel
NMI:          0          0          0          0   Non-maskable interrupts
LOC:      66556      45656      22277      19634   Local timer interrupts
SPU:          0          0          0          0   Spurious interrupts
PMI:          0          0          0          0   Performance monitoring interrupts
IWI:       3765        552        416        338   IRQ work interrupts
RTR:          1          0          0          0   APIC ICR read retries
RES:       1541       1212       1102        713   Rescheduling interrupts
CAL:        949       1092       1070       1171   Function call interrupts
TLB:         79        120        226        131   TLB shootdowns
TRM:          0          0          0          0   Thermal event interrupts
THR:          0          0          0          0   Threshold APIC interrupts
DFR:          0          0          0          0   Deferred Error APIC interrupts
MCE:          0          0          0          0   Machine check exceptions
MCP:          4          4          4          4   Machine check polls
ERR:          0
MIS:          0
PIN:          0          0          0          0   Posted-interrupt notification event
PIW:          0          0          0          0   Posted-interrupt wakeup event
```

可以看到**IRQ0**，**IRQ1**，**IRQ8**和**IRQ9**和Intel 8259A中**提供一样的功能**，**其他的IRQ都没有被分配**。**i8042是键盘中断（PS/2**），有两个是因为**设计时候兼容鼠标**，所以**IRQ1给键盘**，一个**IRQ12给鼠标**。

### 2.1.1 Intel 8259A芯片

Intel 8259A芯片通过2级桥接（slave芯片接在master的IRQ2上），提供总共15个中断。可用的其实更加少，因此在某个阶段（APIC还没出现），Intel提供了IRQ共享的概念，通过添加额外的线路（PIRQ）来使得多个PCI设备使用同一个IRQ随后通过软件的模式来具体区分是哪一个设备触发了中断。单个IRQ最多可以有8个共享（PIRQA - PIRQH）。

![config](./images/1.gif)

注意芯片本身也是有时钟的，默认是。芯片通过引脚INTR来告知CPU有中断到来（CPU通过引脚INTA来反馈信息）；通过D0 - D7来传输数据。芯片的可写寄存器通过IN/OUT指令来操作。

IRQ同时也代表了中断的优先级，数字越小，优先级越高。

### 2.1.2 APIC

伴随着SMP系统的诞生，APIC芯片被引入。APIC包括两种类型芯片：

- local APIC，本地（局部）APIC芯片和CPU Core 一一对应
- I/O APIC，负责链接设备，并发送给local APIC。最多有8个，所有CPU Core共享。

![config](./images/2.gif)

local APIC和I/O APIC之间通过**FSB(系统总线！！！**)相互连接（早期有单独的APIC总线）。

Intel 8259A也可以和APIC桥接：

>In systems containing an 8259 PIC, the 8259 may be connected to the LAPIC in the system's bootstrap processor (BSP), or to one of the system's I/O APICs, or both. Logically, however, the 8259 is only connected once at any given time.

APIC有两种模式：

- 作为一种**标准的 8259A 工作方式**。**Local APIC 被禁止**，外部 **I/O APIC 连接到 CPU**，两条 **LINT0 和 LINT1 分别连接到 INTR 和 NMI 引脚**。
- 作为一种**标准外部 I/O APIC**。**本地 APIC 被激活**，且**所有的外部中断**都通过 **I/O APIC 接收**。

**APIC**提供**更加多的IRQ**，**单个I/O APIC芯片**支持**24个IRQ**。I/O APIC类似于一个**dispatcher**，通过一定的规则，按照**中断的affinity设置**来**派发每个中断**。

同时APIC还提供了IPI（Inter Processor Interrupt）功能，用于发送中断给其他Core，可软件触发。

- send\_IPI\_all() : Sends an IPI to all CPUs (including the sender)
- send\_IPI\_allbutself() : Sends an IPI to all CPUs except the sender
- send\_IPI\_self() : Sends an IPI to the sender CPU
- send\_IPI\_mask() : Sends an IPI to a group of CPUs specified by a bit mask

APIC定义的中断触发的时机有两种：**Edge triggered(边缘触发**） or **Level triggered（水平触发**）

Edge triggered(边缘触发）意味着中断信息的发送是通过电平转换来触发的（**上升沿**或者**下降沿**），也就是类似于脉冲信号。这意味着信号**如果没有被接受就会丢掉**。多个统一中断向量的中断触发会产生多个中断信息。

Level triggered（水平触发）是通过**维持电平来触发**的。系统会**周期性的检查引线的电平**，如果符合要求（比如高电平），那么就认为中断向量被触发。在电平未清除之前，重复触发会被ignore掉（因为一直在高电平）。

准确的说**edge**分为low\-to\-high edge triggered(**上升沿**)和 high\-to\-low edge triggered(**下降沿**)；**level**分为active high level\-sensitive(**高电平维持**)和active low level\-sensitive(**低电平维持**)。具体需要查看**各个芯片的文档**。

查看/proc/interrupts，会发现**两种不同APIC**的设置：apic\-edge 和 apic\-fasteoi。前者如其名字，代表**edge triggered**；后者是**level trigger**，并且**中断的状态(ISR)会保持**，直到**EOI（end of interrupt，代表中断处理完毕**）被**PIC（programmable Interrupt Controller）接收到**，才会**被清除**。

## 2.2 ARM中断子系统

**ARM中断**依赖于**Generic Interrupt Controller（GIC**），每一代都会有不停的演进，支持的中断数目也不同：比如A15的CoreLink GIC-400 Generic Interrupt Controller，支持256个中断向量。具体分区需要参考各个芯片的文档，以下GICv2为例子。（后续的GICv3，GICv4文档ARM没有公布，当然也可以使用其他类型的中断芯片）

GICv2可以管理四类中断类型（CPU内部中断它不管）：

- Peripheral Interrupt（外围中断）
    - Private Peripheral Interrupt (PPI) ：发送给特定CPU的中断
    - Shared Peripheral Interrupt (SPI) ：可以route给任何一个CPU处理的中断
- Software\-generated interrupt (SGI) ：类似APIC的IPI中断，负责Processor之间交互
- Virtual Interrupt：给Virtual Machine使用的，比较少见。
- Maintenance Interrupt：类似于GIC内部的中断，用来enable/disable GIC一些功能。

GICv2指定ID0 - ID31为PPI，ID32往上时SPI，根据芯片类型，甚至支持上千的SPI（虽然A15只有224个）。

ARM没有向前兼容的需求，所以很多时候可以设计的更加激进一些。

另外ARM Core的state要比x86多很多（多达7个），其中有两个专门给interrupt提供的state（知乎表格的功能什么时候能提供呢？！）：

![config](./images/3.jpg)

其中FIQ（快速中断）和IRQ是ARM专门提供来处理中断。FIQ的优先级高于IRQ，并且处理时排斥其他FIQ或者IRQ。

ARM的Vector Table一共7格：

```
Interrupt/Exception/Reset	Mode	Address
Reset				SVC	0x00000000
Undefined instruction		UND	0x00000004
Software interrupt (SWI)	SVC	0x00000008
Prefetch abort			ABT	0x0000000c
Data abort			ABT	0x00000010
Reserved			N/A	0x00000014
IRQ				IRQ	0x00000018
FIQ				FIQ	0x0000001c
```

对于FIQ/IRQ，多个handler通过类似link list的方式串起来，一个中断会按照顺序从头搜索handler，直到找到可以识别和处理的handler。

## 2.3 SMP IRQ affinity

类似于**CPU core的affinity**，中断也可以指定的**绑定**到**一个或者一批特定的core**上执行。

```
/proc/irq/IRQ#/smp_affinity and /proc/irq/IRQ#/smp_affinity_list specify
which target CPUs are permitted for a given IRQ source. It's a bitmask
(smp_affinity) or cpu list (smp_affinity_list) of allowed CPUs. It's not
allowed to turn off all CPUs, and if an IRQ controller does not support
IRQ affinity then the value will not change from the default of all cpus.
```

通过查看/proc/interrupts，如果发现**某个core busying handle 某个中断**，那可以改变这个中断变量的affinity，让其他core分担中断的压力。

不过，SMP IRQ affinity需要kernel和APIC的合作，所以如果可能太老，不一定能够work。这种情况下，请询问是否可以升级kernel。