对Interrupt(中断)和Exception(异常)探讨.

在x86/x64体系中,中断和异常使用相同的执行环境和机制,包括:

① 使用在同一个IDT(Interrupt Descriptor Table)或IVT(Interrupt Vector Table)

② 在同一个Vector范围(0 \~ 255)内

③ 在Handling Interrupt(中断处理)流程上是一致的,当然有些额外区别

**中断**和**异常**事件可以是**主动发起**和**被动产生**的,它们在处理流程中有几个显著区别.

① 对于**硬件**产生的**可屏蔽中断**事件,在**ISR(中断服务例程**)处理完毕后必须**发送EOI(End Of Interrupt**)命令通知**Interrupt controller(中断控制器**)

② 对于**某些异常**,**处理器**会产生相应的**Error Code(错误码**),**Exception处理程序**可以根据情况进行处理

③ **外部**和**内部**的**硬件中断请求**,可以进行**mask(屏蔽**),除了**NMI(不可屏蔽中断**)和一些**由处理器pin产生的中断事件**外(例如: **SMI**)

④ 对于**异常**是**不可屏蔽**的

本章中,将分别使用Interrupt(中断)和Exception(异常)的术语,除非特别说明,否则将互不适用

