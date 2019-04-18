
# 1 简介

使用的dmidecode命令检索任何Linux系统的硬件信息。 假设，如果我们要升级，我们需要收集如内存 ，BIOS和CPU等信息的系统 随着的dmidecode命令的帮助下，我们会知道的细节，而无需打开系统的底盘。 

dmidecode命令适用于RHEL / CentOS的 / Fedora的 / Ubuntu Linux操作系统。

dmidecode允许你在Linux系统下获取有关硬件方面的信息。dmidecode遵循SMBIOS/DMI标准，其输出的信息包括BIOS、系统、主板、处理器、内存、缓存等等。

DMI是英文单词Desktop Management Interface的缩写，也就是桌面管理界面，它含有关于系统硬件的配置信息。计算机**每次启动**时都对**DMI数据进行校验**，如果该数据出错或硬件有所变动，就会**对机器进行检测**，并把测试的数据写入**BIOS芯片保存**。所以如果我们在**BIOS设置**中**禁止了BIOS芯片的刷新功能**或者在**主板使用跳线禁止了 BIOS芯片的刷新功能**，那这台机器的**DMI数据**将**不能被更新**。如果你更换了硬件配置，那么在进行WINDOWS系统时，机器仍旧按老系统的配置进行工作。这样就不能充分发挥新添加硬件的性能，有时还会出现这样或那样的故障。

- DMI（Desktop Management Interface,DMI）就是帮助收集电脑**系统信息**的**管理系统**，DMI信息的收集必须在严格遵照SMBIOS规范的前提下进行。
- SMBIOS（System Management BIOS）是主板或系统制造者以标准格式显示产品管理信息所需遵循的统一规范。

SMBIOS和DMI是由行业指导机构Desktop Management Task Force(DMTF)起草的开放性的技术标准，其中DMI设计适用于任何的平台和操作系统。

DMI充当了管理工具和系统层之间接口的角色。它建立了标准的可管理系统更加方便了电脑厂商和用户对系统的了解。DMI的主要组成部分是Management Information Format(MIF)数据库。这个数据库包括了所有有关电脑系统和配件的信息。通过DMI，用户可以获取序列号、电脑厂商、串口信息以及其它系统配件信息。

dmidecode的作用是将DMI数据库中的信息解码，以可读的文本方式显示。由于DMI信息可以人为修改，因此里面的信息不一定是系统准确的信息。

# 2 命令用法

不带选项执行dmidecode通常会输出所有的硬件信息。dmidecode有个很有用的选项-t，可以指定类型输出相关信息。假如要获得处理器方面的信息，则可以执行：

```
dmidecode -t processor
```

```
Usage: dmidecode [OPTIONS]
```

Options are:

-d：(default:/dev/mem)从设备文件读取信息，输出内容与不加参数标准输出相同。

-h：显示帮助信息。

-s：只显示指定DMI字符串的信息。(string)

-t：只显示指定条目的信息。(type)

-u：显示未解码的原始条目内容。

-- dump-bin FILE: Dump the DMI data to a binary file.

-- from-dump FILE: Read the DMI data from a binary file.

-V：显示版本信息

dmidecode的输出格式一般如下：

```
Handle 0x0002, DMI type 2, 95 bytes.

Base Board Information

     Manufacturer: IBM

     Product Name: Node1 Processor Card

     Version: Not Specified

     Serial Number: Not Specified
```

其中记录头（recode header）包括了：

recode id(Handle)：**DMI表**中的**记录标识符**，这是唯一的，比如上例中的Handle 0x0002.

DMI type id：**记录的类型**，譬如说：BIOS，Memory，上例是type 2，即“Base Board Information”.

recode size：DMI表中对应**记录的大小**，上例为95 bytes。（不包括文本信息，所有实际输出的内容比这个size要更大）。记录头之后就是记录的值。

recoded values：记录值可以是多行的，比如上例显示了主板的制造商（Manufacturer）、Product Name、Version以及Serial Number。

1. 最简单的的显示全部dmi信息

```
[root@localhost ~]# dmidecode
[root@localhost ~]# dmidecode|wc -l
6042
```

2. 更精简的信息显示

```
# dmidecode -q
```

-q(–quite) 只显示必要的信息

3. 显示指定类型的信息：

通常我只想查看某类型，比如CPU，内存或者磁盘的信息而不是全部的。这可以使用\-t(\–type TYPE)来指定信息类型

```
# dmidecode -t bios
# dmidecode -t bios, processor (这种方式不可以用，必须用下面的数字的方式)
# dmidecode -t 0,4 (显示bios和processor)
```

## 2.1 支持的type

可以在man dmidecode里面看到

文本参数支持：

bios, system, baseboard, chassis, processor, memory, cache, connector, slot

bios |  bios的各项信息
-----|-----------
 system |  系统信息，在我的笔记本上可以看到版本、型号、序号等信息。
 baseboard |  主板信息
 chassis |  “底板”，不太理解其含意，期待大家补充
 processor |  CPU的详细信息
 memory |  内存信息，包括目前插的内存条数及大小，支持的单条最大内存和总内存大小等等。
 cache |  缓存信息，似乎是CPU的缓存信息
 connector |  在我的电脑是PCI设备的信息
 slot |  插槽信息


数字参数支持很多：

```
The SMBIOS specification defines the following DMI types:

       Type   Information
       ────────────────────────────────────────────
          0   BIOS
          1   System
          2   Baseboard
          3   Chassis
          4   Processor
          5   Memory Controller
          6   Memory Module
          7   Cache
          8   Port Connector
          9   System Slots
         10   On Board Devices
         11   OEM Strings
         12   System Configuration Options
         13   BIOS Language

         14   Group Associations
         15   System Event Log
         16   Physical Memory Array
         17   Memory Device
         18   32-bit Memory Error
         19   Memory Array Mapped Address
         20   Memory Device Mapped Address
         21   Built-in Pointing Device
         22   Portable Battery
         23   System Reset
         24   Hardware Security
         25   System Power Controls
         26   Voltage Probe
         27   Cooling Device
         28   Temperature Probe
         29   Electrical Current Probe
         30   Out-of-band Remote Access
         31   Boot Integrity Services
         32   System Boot
         33   64-bit Memory Error
         34   Management Device
         35   Management Device Component
         36   Management Device Threshold Data
         37   Memory Channel
         38   IPMI Device
         39   Power Supply
         40   Additional Information
         41   Onboard Devices Extended Information
         42   Management Controller Host Interface
```

4. 通过关键字查看信息：

比如只想查看系统序列号，可以使用:

```
# dmidecode -s system-serial-number
218480676
```

-s (–string keyword)支持的keyword包括：

bios-vendor,bios-version, bios-release-date,

system-manufacturer, system-product-name, system-version, system-serial-number, baseboard-manu-facturer,baseboard-product-name, baseboard-version, baseboard-serial-number, baseboard-asset-tag, chassis-manufacturer, chas-sis-version, chassis-serial-number, chassis-asset-tag, processor-manufacturer, processor-version.

# 3 查看内存信息和支持的最大内存

查看当前内存和支持的最大内存

## 3.1 查看当前物理内存

Linux下，可以使用free或者查看meminfo来获得当前的物理内存：

```
[root@compute1 ~]# free
              total        used        free      shared  buff/cache   available
Mem:      263923064     3228100   257267132       18036     3427832   259021016
Swap:       4194300           0     4194300
[root@compute1 ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:           251G        3.1G        245G         17M        3.3G        247G
Swap:          4.0G          0B        4.0G
```

显示当前服务器物理内存是256G

## 3.2 硬件支持的信息

服务器到底能扩展到多大的内存?

```
# 16代表物理内存数组
[root@compute1 ~]# dmidecode -t 16
# dmidecode 3.0
Getting SMBIOS data from sysfs.
SMBIOS 3.0 present.

Handle 0x0066, DMI type 16, 23 bytes
Physical Memory Array
	Location: System Board Or Motherboard
	Use: System Memory
	Error Correction Type: Multi-bit ECC
	Maximum Capacity: 384 GB (可扩展到384GB)
	Error Information Handle: Not Provided
	Number Of Devices: 6

Handle 0x0070, DMI type 16, 23 bytes
Physical Memory Array
	Location: System Board Or Motherboard
	Use: System Memory
	Error Correction Type: Multi-bit ECC
	Maximum Capacity: 384 GB (可扩展到384GB)
	Error Information Handle: Not Provided
	Number Of Devices: 6

Handle 0x007A, DMI type 16, 23 bytes
Physical Memory Array
	Location: System Board Or Motherboard
	Use: System Memory
	Error Correction Type: Multi-bit ECC
	Maximum Capacity: 384 GB (可扩展到384GB)
	Error Information Handle: Not Provided
	Number Of Devices: 6

Handle 0x0084, DMI type 16, 23 bytes
Physical Memory Array
	Location: System Board Or Motherboard
	Use: System Memory
	Error Correction Type: Multi-bit ECC
	Maximum Capacity: 384 GB (可扩展到384GB)
	Error Information Handle: Not Provided
	Number Of Devices: 6
```

但是，事实不一定如此，因为插槽可能已经插满了。也就是我们还必须查清这里的256G到底是4\*2GB, 2\*4GB还是其他？

从上面信息可以看到:

- 内存插槽数: 
