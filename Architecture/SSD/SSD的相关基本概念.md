[TOC]

**SSD**拥有比**HDD**更快的读写速度，但**SATA总线标准**却**拖累了SSD性能**的发挥。越来越多的笔记本都配备了**支持PCI\-E总线标准的M.2插槽**，这就让**更高速的NVMe SSD**有了用武之地。

啥叫PCI-E 3.0×4（标准名称为PCIExpress Gen 3×4）？NVMe又是什么？M.2接口不是SATA总线吗？

# 1 硬盘接口/插槽类型

IDE/SATA/MSATA/eSATA/SATA-E/M.2接口/插槽类型, 用于连接硬盘的插槽

- 早期的IDE接口, 全称 Integrated Drive Electronics，即“电子集成驱动器”，俗称PATA并口, 是2000年以前主流的硬盘传输接口

![config](./images/3.png)

- 主流的硬盘接口是SATA接口, 区分SATA2.0和3.0, 目前常见的固态硬盘, 就是SATA 3.0, 5Gbps的速度. 这已经是SATA瓶颈, 以后不会出现SATA 3.1的

机械: 厚, 内置碟片, 读写运行有声音

固态: 内置闪存颗粒, 没有碟片, 轻, 小

![config](./images/4.png)

![config](./images/5.png)

- mSATA也就是microSATA, 用于老式笔记本硬盘存储, 有点像M.2, 外观上无法区分

![config](./images/6.png)

mSATA接口是标准SATA的迷你版本，通过mini PCI-E界面传输信号，传输速度支持1.5Gbps、3Gbps、6Gbps三种模式。

- ESATA这个接口，目前只能在少部分的老式笔记本上能看到，也是鸡肋的一种，研发出来是为了替代USB 2.0的，没想到直接被后面出来的USb 3.0干掉。

![config](./images/7.png)

- 鸡肋的设置SATA-E，10Gbps的速度。家用消费级市场还未普及，目前只在工业市场看到过。

![config](./images/8.png)

- m.2是一种固态硬盘新接口，是Intel推出的一种替代MSATA新的接口规范，也就是我们以前经常提到的NGFF，英文全称为：Next转载自电脑百事网 Generation Form Factor. 可以将m.2理解为比**3.5英寸**和**2.5英寸**更小的**硬盘**和硬盘**插槽标准**.

需要注意的是, M.2接口(SSD上的金手指形状)和插槽(主板上)又被细分为B key(又称socket2)和M key(socket3)两个模组, 二者由于金手指缺口和针脚数量不同而产生兼容问题.

![config](./images/9.png)

![config](./images/10.png)

![config](./images/11.png)

目前固态硬盘（SSD）常用的接口主要有3种：

① SATA3 - 外形尺寸是2.5寸硬盘的标准尺寸，与2.5寸机械硬盘可以互换。

② mSATA - 接口尺寸与笔记本内置无线网卡相同，不过mSATA SSD的外形尺寸比无线网卡更大。

③ M.2 - 初期称为NGFF接口，是近两年新出的接口，为专门为超小尺寸而设计的，使用PCI-E通道，体积更小，速度更快。

![config](./images/1.png)

m.2的SSD比较小, m.2最早还用过NGFF名字, 它有很多长度版本, 最常见是42/60/80mm三种

![config](./images/2.png)

![config](./images/12.jpg)

不仅仅是长度差异, 还存在通道, 协议, 兼容性等差异

M.2接口固态硬盘又分为：SATA和PCI\-E两种，虽说长得一模一样，但性能却是差之千里。

# 2 总线类型

PCI\-e/SATA

M.2 SSD通常会伴随一个名叫PCI\-E的参数, 这是啥?

M.2固态硬盘还分通道，目前M.2固态硬盘分PCI\-E与SATA**两种通道**, 也就是总线类型，两者在性能、价格等方面也存在明显区别.

![config](./images/13.png)

上图为128GB与256GB M.2固态硬盘外观对比

- 为什么同为M.2接口固态硬盘还会有PCI\-E和SATA之分？

这是因为两者所走的通道不同，M.2有两种接口定义：Socket 2和Socket 3。

Socket 2支持SATA、PCI\-E X2通道的SSD，Socket 3专为高性能存储设计，支持PCI\-E X4。由于走的路不一样，性质也就截然不同了。

这两种总线标准直接决定了M.2 SSD的读写速度.

注: PCI\-E 3.0X4完整叫法应该是PCI Express Gen3X4

# 3 协议标准

AHCI/NVMe

使用**SATA**硬盘装系统时, 需要进入**BIOS开启名叫AHCI的选项**. 这个AHCI就是**SATA串口硬盘对应的协议标准(逻辑设备接口标准**), 可以视为是**SATA的优化驱动**.

**NVme是AHCI的进阶版**, 也是一种协议标准, 属于针对**PCI\-E总线SSD定制的一种高速协议**(可以视为驱动程序)

注: 目前支持NVMe协议的M.2 SSD一定采用了PCI\-E 3.0X4总线标准, 但采用PCI\-E 3.0X4总线的M.2 SSD却并不一定支持NVMe协议

# 4 兼容问题

M.2 SSD的兼容性, 就是针对前面的插槽/接口类型, 总线类型和协议标准展开的

![config](./images/14.png)

![config](./images/15.png)