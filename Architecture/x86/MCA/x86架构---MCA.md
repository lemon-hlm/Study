
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
	* [1.1 不可纠正的MCE(uncorrected machine\-check error)](#11-不可纠正的mceuncorrected-machine-check-error)
	* [1.2 可纠正的MCE(corrected machine\-check error)](#12-可纠正的mcecorrected-machine-check-error)
	* [1.3 额外功能](#13-额外功能)
* [2 Machine Check MSR](#2-machine-check-msr)
	* [2.1 Machine\-Check Global Control MSRs](#21-machine-check-global-control-msrs)
		* [2.1.1 IA32\_MCG\_CAP MSR](#211-ia32_mcg_cap-msr)
		* [2.1.2 IA32\_MCG\_STATUS MSR](#212-ia32_mcg_status-msr)
		* [2.1.3 IA32\_MCG\_CTL MSR](#213-ia32_mcg_ctl-msr)
		* [2.1.4 IA32\_MCG\_EXT\_CTL MSR](#214-ia32_mcg_ext_ctl-msr)
		* [2.1.5 Enabling Local Machine Check](#215-enabling-local-machine-check)
	* [2.2 错误报告寄存器组(Error\-Reporting Register Banks)](#22-错误报告寄存器组error-reporting-register-banks)
		* [2.2.1 IA32\_MCi\_CTL MSRs](#221-ia32_mci_ctl-msrs)
		* [2.2.2 IA32\_MCi\_STATUS MSRS](#222-ia32_mci_status-msrs)
		* [2.2.3 IA32\_MCi\_ADDR MSRs](#223-ia32_mci_addr-msrs)
		* [2.2.4 IA32\_MCi\_MISC MSRs](#224-ia32_mci_misc-msrs)
		* [2.2.5 IA32\_MCi\_CTL2 MSRs](#225-ia32_mci_ctl2-msrs)
* [3 CMCI](#3-cmci)
* [4 MCA的初始化](#4-mca的初始化)
* [5 MSR的读写](#5-msr的读写)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 概述

Intel从奔腾4开始的CPU中增加了一种机制，称为MCA——Machine Check Architecture，它用来**检测硬件**（这里的Machine表示的就是硬件）错误，比如系统总线错误、ECC错误、奇偶校验错误、缓存错误、TLB错误等等。不仅硬件故障会引起MCE，不恰当的BIOS配置、firmware bug、软件bug也有可能引起MCE。

这套系统通过**一定数量的MSR**（Model Specific Register）来实现，这些MSR分为两个部分，一部分用来**进行设置**，另一部分用来**描述发生的硬件错误**。

## 1.1 不可纠正的MCE(uncorrected machine\-check error)

当CPU检测到**不可纠正的MCE（Machine Check Error**）时，就会触发\#**MC**（**Machine Check Exception**, 中断号是十进制18），通常**软件**会**注册相关的函数**来处理\#MC，在这个函数中会通过读取MSR来收集MCE的错误信息，但是不被允许重启处理器。

当然由于发生的**MCE**可能是**非常致命**的，**CPU直接重启**了，没有办法完成MCE处理函数；甚至有可能在MCE处理函数中又触发了不可纠正的MCE，也会导致系统直接重启。

发现硬件错误时触发的异常(exception)，中断号是18，异常的类型是abort：

![](./images/2019-04-28-15-02-46.png)

## 1.2 可纠正的MCE(corrected machine\-check error)

从CPUID的DisplayFamily\_DisplayModel为06H\_1AH开始, CPU可以报告可纠正的机器检查错误信息, 并为软件提供可编程中断来响应MC错误, 称为可纠正机器检查错误中断(CMCI). 

CPU检测到**可纠正的MCE**，当可纠正的MCE数量**超过一定的阈值**时，会触发**CMCI（Corrected Machine Check Error Interrupt**），此时软件可以捕捉到该中断并进行相应的处理。

CMCI是在**MCA之后才加入**的，算是对MCA的一个增强，在此之前**软件只能通过轮询可纠正MCE相关的MSR**才能实现相关的操作。

Corrected machine-check error interrupt (CMCI) 是MCA的增强特性。在原来的芯片里面，都是使用一种叫做threshold-based error reporting的机制来处理corrected error. 但是threshold-based error reporting需要系统软件周期性的轮询检测硬件的corrected MC errors，造成CPU的浪费。 CMCI 提供了一种机制，当corrected error发生侧次数到达阀值的时候，就会发送一个信号给本地的CPU来通知系统软件。当然，系统软件可以通过IA32\_MCi\_CTL2 MSRs来控制该特性的开关

## 1.3 额外功能

支持**机器检查架构**和**CMCI**的**英特尔64处理器**还可以支持**额外的增强功能**，即支持从**某些不可纠正**的**可恢复机器检查错误**中进行**软件恢复**。

# 2 Machine Check MSR

![](./images/2019-04-28-14-30-05.png)

上图基本包含了MCA相关的所有MSR。

它分为左右两个部分，左边的是全局的寄存器，右边表示的是多组寄存器。

i表示的是各个组的Index。这里的组有一个称呼是Error Reporting Register Bank。

MCA通过若干Bank的MSR寄存器来表示各种类型的MCE。

下面简单介绍一下这些寄存器。

## 2.1 Machine\-Check Global Control MSRs

机器检查全局控制MSR包括IA32\_MCG\_CAP，IA32\_MCG\_STATUS，以及可选的IA32\_MCG\_CTL和IA32\_MCG\_EXT\_CTL。

### 2.1.1 IA32\_MCG\_CAP MSR

这个MSR描述了**当前CPU处理MCA的能力**，机器检查体系结构的信息, 具体每个位的作用如下所示：

![](./images/2019-04-28-14-31-53.png)

BIT0-7：表示的是特定CPU支持可用的**硬件单元错误报告库的个数(hardware unit error-reporting banks**)；

BIT8：1表示**IA32\_MCG\_CTL**有效，如果是**0的话表示无效**，读取该IA32\_MCG\_CTL这个MSR可能发生Exception（至少在UEFI下是这样）；

BIT9：1表示**IA32\_MCG\_EXT\_CTL**有效，反之无效，这个与BIT8的作用类似；

BIT10：1表示**支持CMCI**，但是CMCI是否能用还需要通过**IA32\_MCi\_CTL2**这个MSR的**BIT30来使能**；

BIT11：1表示IA32\_MCi\_STATUS这个MSR的BIT56\-55是保留的，BIT54-53是用来上报Threshold-based Error状态的；

BIT16-23：表示存在的Extended Machine Check State寄存器的个数；

BIT24：1表示CPU支持Software Error Recovery；

BIT25：1表示CPU支持**增强版的MCA**；

BIT26：1表示支持更多的错误记录（需要UEFI、ACPI的支持）；

BIT27：1表示支持Local Machine Check Exception；

### 2.1.2 IA32\_MCG\_STATUS MSR

该MSR记录了**MCE发生时CPU的状态**，主要的BIT位介绍如下：

![](./images/2019-04-28-14-34-21.png)

- Bit 0: Restart IP Valid. 表示程序的执行是否可以在被异常中断的指令处重新开始。
- Bit 1: Error IP Valid. 表示被中断的指令是否与MCE错误直接相关。
- Bit 2: Machine Check In Progress. 表示 machine check 正在进行中。
- bit 3: 设置后说明生成本地machine\-check exception. 这表示当前的机器检查事件仅传递给此逻辑处理器。

### 2.1.3 IA32\_MCG\_CTL MSR

这个寄存器的存在依赖于IA32_MCG_CAP这个MSR的BIT8。

这个寄存器主要用来Disable（写1）或者Enable（写全0）**MCA功能**。

### 2.1.4 IA32\_MCG\_EXT\_CTL MSR

这个寄存器同样依赖于IA32\_MCA\_CAP这个MSR，这次依赖的是BIT9。

该MSR的BIT位说明如下图所示：

![](./images/2019-04-28-15-41-55.png)

目前有就BIT0有用，用来Disable（写1）或者Enable（写0）**LMCE**，这个LMCE的功能就是使**硬件**能够将**某些MCE**发送给**单个的逻辑处理器**。

### 2.1.5 Enabling Local Machine Check

LMCE的预期用途需要平台软件和系统软件的正确配置。 平台软件可以通过设置IA32\_FEATURE\_CONTROL MSR（MSR地址3AH）中的位20（LMCE\_ON）来打开LMCE。

系统软件必须确保在尝试设置IA32\_MCG_EXT_CTL.LMCE_EN（位0）之前设置IA32_FEATURE_CONTROL.Lock（位0）和IA32_FEATURE_CONTROL.LMCE_ON（位20）。 

当系统软件**启用LMCE**时，**硬件**将确定**是否只能将特定错误**传递给**单个逻辑处理器**。 软件不应假设硬件可以选择作为LMCE提供的错误类型。

## 2.2 错误报告寄存器组(Error\-Reporting Register Banks)

以上都是全局的MSR.

**每个错误报告寄存器库**可以包含IA32\_MCi\_CTL，IA32\_MCi\_STATUS，IA32\_MCi\_ADDR和IA32\_MCi\_MISC MSR。报告库的数量由IA32\_MCG\_CAP MSR（地址0179H）的位\[7：0]表示。第一个错误报告寄存器（IA32\_MC0\_CTL）**始终从地址400H开始**。

有关Pentium 4，Intel Atom和Intel Xeon处理器中错误报告寄存器的地址，请参阅“英特尔®64和IA-32架构软件开发人员手册”第4卷第2章“特定于型号的寄存器（MSR）”。以及错误报告寄存器P6系列处理器的地址。

这些寄存器的第一个是IA32\_MC0\_CTL，它的地址一般都是400H。

之后接着的是IA32\_MC0\_STATUS，IA32\_MC0\_ADDR，IA32\_MC0\_MISC，但是在之后并不是IA32\_MC0\_CTL2，而是IA32\_MC1\_CTL；对于IA32\_MCi\_CTL2来说，它的地址跟上面的这些不在一起，第一个IA32\_MC0\_CTL2是在280H，之后是IA32\_MC1\_CTL2在281H，以此类推。

### 2.2.1 IA32\_MCi\_CTL MSRs

IA32\_MCi\_CTL MSR控制\#MC的信号，以发现由特定硬件单元（或硬件单元组）产生的错误。

每个Bank的CTL的作用是用来控制在发生**哪些MCA**的时候来**触发\#MC**：

![](./images/2019-04-28-20-05-55.png)

这里的64个BIT位，设置某个BIT位就会使对应BIT位的**MCA类型在发生**时**触发\#MC**。

### 2.2.2 IA32\_MCi\_STATUS MSRS

这类MSR的作用就是显示MCE信息：

![](./images/2019-04-28-20-08-42.png)

注意只有当VAL这个BIT位（BIT63）为1时才表示发生了对应这个Bank的MCE。当MCE发生了，软件需要给这个VAL位写0来清零（如果有可能的话，因为对于不可纠正的MCE可能软件会 来不及写），不能往这位写1，会出现Exception。

BIT0-15，BIT16-31：这个两个部分都表示MCE的错误类型，前者是通用的，后者是跟CPU有关的；

BIT58：1表示IA32_MCi_ADDR这个MSR是有效的，反之无效；

BIT59：1表示IA32_MCi_MISC这个MSR是有效的，反之无效；这两个BIT是因为不同MCE错误并不是都需要ADDR和MSIC这样的MSR；

BIT60：这个位于IA32_MCi_CTL中的位是对应的，那边使能了，这里就是1；

BIT61：表示MCE是不可纠正的；

BIT62：表示发生了二次的MCE，这个时候到底这个Bank表示的是哪一次的MCE信息，需要根据一定的规则来确定：

![](./images/2019-04-28-20-09-12.png)

其它寄存器不介绍了, 详细看手册

### 2.2.3 IA32\_MCi\_ADDR MSRs

![](./images/2019-04-28-20-10-20.png)

这个地址指向**内存中导致MCE的代码或者数据**。

注意这个地址在不同的内存模型下可以是偏移地址，虚拟地址和物理地址中的一种，这个需要MISC这个MSR来确定，下面会讲到。

这个MSR也可以手动清零，写1会出错。

### 2.2.4 IA32\_MCi\_MISC MSRs

这个寄存器的BIT位说明如下：

![](./images/2019-04-28-20-11-23.png)

这里的Address Mode说明如下：

![](./images/2019-04-28-20-11-35.png)

### 2.2.5 IA32\_MCi\_CTL2 MSRs

这个寄存器就是为CMCI使用的，BIT位说明如下：

![](./images/2019-04-28-20-12-02.png)

一个是用于使能CMCI，另一个是用来设置CMCI的阈值。

除了上述的MSR之外，在IA32_MCG_CAP这个MSR的说明中还提到过它的BIT16-23还提到了额外的MSR，它们称为Extended Machine Check State，这些MSR的描述如下：

![](./images/2019-04-28-20-12-19.png)

上图实际上只展示了非64位CPU的MSR，还有一个64位CPU的MSR，这里就不再多说。



需要注意，实际上上面的这些寄存器并不需要自己一个个去对比和解析，Intel提供了一个工具叫做**MCE Decoder**，可以用来**解析MCE**。

另外在Intel的开发者手册中有专门的一个章节解析MCE错误：《CHAPTER 16 INTERPRETING MACHINE-CHECK ERROR CODES》。

# 3 CMCI

前面以及提到，CMCI是后期加入到MCA的一种机制，它将**错误上报的阈值操作**从原始的**软件轮询**变成了**硬件中断触发**。

一个CPU**是否支持CMCI**需要查看**IA32\_MCG\_CAP**的BIT10，如果该位是1就表示支持。

另外**CMCI默认是关闭的**，需要通过IA32\_MCi\_CTL2的BIT30来打开，并设置BIT0-14的阈值，注意 **每个Bank都要设置！！！**。

设置的时候首先**写1**到**IA32\_MCi\_CTL2**的**BIT30**，再**读取**这个值，如果值变成了1，说明CMCI使能了，否则就是CPU不支持CMCI；之后再写阈值到BIT0-14，如果读出来的值是0，表示不支持阈值，否则就是成功设置了阈值。

CMCI是通过Local ACPI来实现的，具体的示意图如下：

![](./images/2019-04-28-20-14-25.png)

在**Local ACPI Table**中有专门处理CMCI的寄存器，称为**LVT CMCI Register (FEE0 02F0H**)：

![](./images/2019-04-28-20-14-38.png)

BIT0-7：中断向量；

BIT8-10：Delivery Mode，比如SMI，NMI等；

BIT12：Delivery Status，0表示没有中断，1表示中断正在发生；

BIT17：Interrupt Mask，0表示接收中断，1表示屏蔽中断；

关于CMCI的初始化和CMCI处理函数的实现，手册上有部分的介绍，不过没有什么源代码可以借鉴，这个不展开了。

# 4 MCA的初始化

手册上有一个伪代码可供参考

```
IF CPU supports MCE
THEN
	IF CPU supports MCA
	THEN
		IF (IA32_MCG_CAP.MCG_CTL_P = 1)
		(* IA32_MCG_CTL register is present *)
		THEN
			IA32_MCG_CTL ← FFFFFFFFFFFFFFFFH;
			(* enables all MCA features *)
		FI
		IF (IA32_MCG_CAP.MCG_LMCE_P = 1 and IA32_FEATURE_CONTROL.LOCK = 1 and IA32_FEATURE_CONTROL.LMCE_ON= 1)
		(* IA32_MCG_EXT_CTL register is present and platform has enabled LMCE to permit system software to use LMCE *)
		THEN
			IA32_MCG_EXT_CTL ← IA32_MCG_EXT_CTL | 01H;
			(* System software enables LMCE capability for hardware to signal MCE to a single logical processor*)
		FI
		(* Determine number of error-reporting banks supported *)
		COUNT← IA32_MCG_CAP.Count;
		MAX_BANK_NUMBER ← COUNT - 1;
		IF (Processor Family is 6H and Processor EXTMODEL:MODEL is less than 1AH)
		THEN
			(* Enable logging of all errors except for MC0_CTL register *)
			FOR error-reporting banks (1 through MAX_BANK_NUMBER)
				DO
					IA32_MCi_CTL ← 0FFFFFFFFFFFFFFFFH;
				OD
		ELSE
			(* Enable logging of all errors including MC0_CTL register *)
			FOR error-reporting banks (0 through MAX_BANK_NUMBER)
				DO
					IA32_MCi_CTL ← 0FFFFFFFFFFFFFFFFH;
				OD
		FI
		(* BIOS clears all errors only on power-on reset *)
		IF (BIOS detects Power-on reset)
		THEN
			FOR error-reporting banks (0 through MAX_BANK_NUMBER)
				DO
					IA32_MCi_STATUS ← 0;
				OD
		ELSE
			FOR error-reporting banks (0 through MAX_BANK_NUMBER)
				DO
					(Optional for BIOS and OS) Log valid errors
					(OS only) IA32_MCi_STATUS ← 0;
				OD
		FI
	FI
	Setup the Machine Check Exception (#MC) handler for vector 18 in IDT
	Set the MCE bit (bit 6) in CR4 register to enable Machine-Check Exceptions
FI
```

# 5 MSR的读写

x86平台读写MSR有专门的指令，分别是rdmsr和wrmsr。下面是MSR读写的一个基本实现：

```cpp
/**
  Returns a 64-bit Machine Specific Register(MSR).
  Reads and returns the 64-bit MSR specified by Index. No parameter checking is
  performed on Index, and some Index values may cause CPU exceptions. The
  caller must either guarantee that Index is valid, or the caller must set up
  exception handlers to catch the exceptions. This function is only available
  on IA-32 and X64.
  @param  Index The 32-bit MSR index to read.
  @return The value of the MSR identified by Index.
**/
UINT64
EFIAPI
AsmReadMsr64 (
  IN      UINT32                    Index
  )
{
  UINT32 LowData;
  UINT32 HighData;
  
  __asm__ __volatile__ (
    "rdmsr"
    : "=a" (LowData),   // %0
      "=d" (HighData)   // %1
    : "c"  (Index)      // %2
    );
    
  return (((UINT64)HighData) << 32) | LowData;
}
 
/**
  Writes a 64-bit value to a Machine Specific Register(MSR), and returns the
  value.
  Writes the 64-bit value specified by Value to the MSR specified by Index. The
  64-bit value written to the MSR is returned. No parameter checking is
  performed on Index or Value, and some of these may cause CPU exceptions. The
  caller must either guarantee that Index and Value are valid, or the caller
  must establish proper exception handlers. This function is only available on
  IA-32 and X64.
  @param  Index The 32-bit MSR index to write.
  @param  Value The 64-bit value to write to the MSR.
  @return Value
**/
UINT64
EFIAPI
AsmWriteMsr64 (
  IN      UINT32                    Index,
  IN      UINT64                    Value
  )
{
  UINT32 LowData;
  UINT32 HighData;
 
  LowData  = (UINT32)(Value);
  HighData = (UINT32)(Value >> 32);
  
  __asm__ __volatile__ (
    "wrmsr"
    :
    : "c" (Index),
      "a" (LowData),
      "d" (HighData)
    );
    
  return Value;
}
```

上面是GCC版本的，还有汇编版本的：

```assembly
;-------------------------------------------------------
; UINT64
; EFIAPI
; AsmReadMsr64 (
;   IN UINT64  Index
;   );
;-------------------------------------------------------
AsmReadMsr64    PROC
    mov     ecx, [esp + 4]
    rdmsr
    ret
AsmReadMsr64    ENDP
 
;-------------------------------------------------------
; UINT64
; EFIAPI
; AsmWriteMsr64 (
;   IN UINT32  Index,
;   IN UINT64  Value
;   );
;--------------------------------------------------------
AsmWriteMsr64   PROC
    mov     edx, [esp + 12]
    mov     eax, [esp + 8]
    mov     ecx, [esp + 4]
    wrmsr
    ret
AsmWriteMsr64   ENDP
```

# 参考

- x86架构——MCA: https://blog.csdn.net/jiangwei0512/article/details/62456226
- Intel手册卷3第15章(CHAPTER 15 MACHINE-CHECK ARCHITECTURE)、16章(CHAPTER 16 INTERPRETING MACHINE-CHECK ERROR CODES)
- 怎么诊断MACE: http://linuxperf.com/?p=105
- MCA机制：硬件错误检测架构: https://blog.csdn.net/chengm8/article/details/53003134