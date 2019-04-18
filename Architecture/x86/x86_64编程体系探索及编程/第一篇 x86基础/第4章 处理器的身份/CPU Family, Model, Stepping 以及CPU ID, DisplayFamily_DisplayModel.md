
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [CPU ID](#cpu-id)
	* [Type（类型）](#type类型)
	* [Family（系列）](#family系列)
	* [Mode（型号）](#mode型号)
	* [Stepping（步进编号）](#stepping步进编号)
	* [Brand ID（品种标识）](#brand-id品种标识)

<!-- /code_chunk_output -->

![config](./images/26.png)

/proc/cpuinfo文件的内容包括有：

```
vendor_id : GenuineIntel   
cpu family: 6    
model     : 23
model name: Intel(R) Core(TM)2 Quad CPU    Q9400  @ 2.66GHz
stepping  : 10
```

这里面

**vendor\_id**: **CPU制造商**
    
**cpu family**: CPU产品**系列代号**。此分类标识英特尔**微处理器的品牌**以及**属于第几代产品**。例如，当今的P6系列(第六代)英特尔微处理器包括英特尔Celeron、Pentium II、Pentium II Xeon、Pendum IⅡ和Pentium III Xeon处理器。

- “1”表示为8086和80186级芯片；
- “2”表示为286级芯片；
- “3”表示为386级芯片； 
- “4”表示为486级芯片(SX、DX、：DX2、DX4)； 
- “5”表示为P5级芯片(经典奔腾和多能奔腾)；
- “6”表示为P6级芯片(包括Celeron、PentiumII、PenfiumIII系列)； 
- “F”代表奔腾Ⅳ。 

**model**: CPU属于**其系列**中的**哪一代的代号**。“型号”编号可以让英特尔识别微处理器的制造技术以及属于**第几代设计**(例如型号4)。**型号与系列**通常是**相互配合使用**的，用以确定您的计算机中所安装的处理器是属于处理器系列中的哪一种特定类型。在与英特尔联系时，此信息通常用以识别特定的处理器。

- “1”为Pentium Pro(高能奔腾)；
- “2”为Pentium Pro(高能奔腾)；
- “3”为Klamath(Pentium II)；
- “4”为Deschutes(Pentium II)；
- “5”为Covington(Celeron)； 
- “6”为Mendocino(Celeron A)； 
- “7”为Katmai(Penfium III)；
- “8”为Coppermine(Penfium III)

**model name**: CPU属于的**名字**及其**编号**、标称**主频**

**stepping**: CPU属于**制作更新版本**。Stepping ID(步进)也叫分级鉴别产品数据转换规范， “步进”编号标识生产英特尔微处理器的**设计或制造版本数据**(例如步进4)。步进用于标识一次“**修订**”，通过使用唯一的步进，可以有效地控制和跟踪所做的更改。步进还可以让最终用户**更具体地识别其系统所安装的处理器版本**。在尝试确定微处理器的内部设计或制造特性时，英特尔可能会需要使用此分类数据。
        
- Katmai Stepping含义：“2”为kB0步进；“3”为kC0步进。
- Coppermine Stepping含义：“l”为cA2步进；“3”为cB0步进；“6”为cC0步进。

# CPU ID

CPU ID是CPU生产厂家为识别不同类型的CPU，而为CPU制订的不同的单一的代码；不同厂家的CPU，其CPU ID定义也是不同的；如“0F24”（Inter处理器）、“681H”（AMD处理器），根据这些数字代码即可判断CPU属于哪种类型，这就是一般意义上的CPU ID。 

由于计算机使用的是十六进制，因此CPUID也是以十六进制表示的。Inter处理器的CPU ID一共包含四个数字，如“0F24”，从左至右分别表示 Type（类型）、Family（系列）、Mode（型号）和Stepping（步进编号）。从CPUID为“068X”的处理器开始，Inter另外增 加了Brand ID（品种标识）用来辅助应用程序识别CPU的类型，因此根据“068X”CPUID还不能正确判别Pentium和Celerom处理 器。必须配合Brand ID来进行细分。AMD处理器一般分为三位，如“681”，从左至右分别表示为Family（系列）、Mode（型号）和 Stepping（步进编号）。 

## Type（类型） 

类型标识用来区别INTEL微处理器是用于由最终用户安装，还是由专业个人计算机系 统集成商、服务公司或制作商安装；数字“1”标识所测试的微处理器是用于由用户安装的；数字“0”标识所测试的微处理器是用于由专业个人计算机系统集成 商、服务公司或制作商安装的。我们通常使用的INTEL处理器类型标识都是“0”，“0F24”CPUID就属于这种类型。 

## Family（系列） 

系列标识可用来确定处理器属于那一代产品。如6系列的INTEL处理器包括Pentium Pro、Pentium II、Pentium II Xeon、Pentium III和Pentium III Xeon处理器。5系列（第五代）包括Pentium处理器和采用 MMX技术的Pentium处理器。AMD的6系列实际指有K7系列CPU，有DURON和ATHION两大类。最新一代的INTEL Pentium 4系列处理器（包括相同核心的Celerom处理器）的系列值为“F” 

## Mode（型号） 

型号标识可用来 确定处理器的制作技术以及属于该系列的第几代设计（或核心），型号与系列通常是相互配合使用的，用于确定计算机所安装的处理器是属于某系列处理器的哪种特 定类型。如可确定Celerom处理器是Coppermine还是Tualutin核心；Athlon XP处理器是Paiomino还是 Thorouhgbred核心。 

## Stepping（步进编号） 

步进编号用来标识处理器的设计或制作版本，有助于控制和跟踪处理器的更改，步进还可以让最终用户更具体地识别其系统安装的处理器版本，确定微处理器的内部设计或制作特性。步进编号就好比处理器的小版本号，如CPUID为“686”和“686A”就好比WINZIP8.0和8.1的关系。步进编号和核心步进是密切联系的。如CPUID为“686”的Pentium III 处理器是cCO核心，而“686A”表示的是更新版本cD0核心。 

## Brand ID（品种标识） 

INTEL从Coppermine核心的处理器开始引入Brand ID作为CPU的辅助识别手段。如我们通过Brand ID可以识别出处理器究竟是Celerom还是Pentium 4。


如上的计算机的CPUID为 7A 06 01 00 FF FB EB BF.

而它对应的DisplayFamily\_DisplayModel为，06\_17H, 因为十六进制的17为23，详细内容参见：Intel(R) 64 and IA-32 Architectures Software Developer’s Manual Volume 3 (3A, 3B & 3C): System Programming Guide中 CHAPTER 35 MODEL-SPECIFIC REGISTERS (MSRS) Table 35-1. CPUID Signature Values of DisplayFamily\_DisplayModel