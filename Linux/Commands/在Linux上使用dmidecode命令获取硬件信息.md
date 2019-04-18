
# 1 简介

使用的dmidecode命令检索任何Linux系统的硬件信息。 假设，如果我们要升级，我们需要收集如内存 ，BIOS和CPU等信息的系统 随着的dmidecode命令的帮助下，我们会知道的细节，而无需打开系统的底盘。 

dmidecode命令适用于RHEL / CentOS的 / Fedora的 / Ubuntu Linux操作系统。

dmidecode允许你在Linux系统下获取有关硬件方面的信息。dmidecode遵循SMBIOS/DMI标准，其输出的信息包括BIOS、系统、主板、处理器、内存、缓存等等。

DMI（Desktop Management Interface,DMI）就是帮助收集电脑系统信息的管理系统，DMI信息的收集必须在严格遵照SMBIOS规范的前提下进行。SMBIOS（System Management BIOS）是主板或系统制造者以标准格式显示产品管理信息所需遵循的统一规范。SMBIOS和DMI是由行业指导机构Desktop Management Task Force(DMTF)起草的开放性的技术标准，其中DMI设计适用于任何的平台和操作系统。

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

recode id(Handle)：DMI表中的记录标识符，这是唯一的，比如上例中的Handle 0x0002.

DMI type id：记录的类型，譬如说：BIOS，Memory，上例是type 2，即“Base Board Information”.

recode size：DMI表中对应记录的大小，上例为95 bytes。（不包括文本信息，所有实际输出的内容比这个size要更大）。记录头之后就是记录的值。

recoded values：记录值可以是多行的，比如上例显示了主板的制造商（Manufacturer）、Product Name、Version以及Serial Number。

1. 最简单的的显示全部dmi信息

```
[root@localhost ~]# dmidecode
[root@localhost ~]# dmidecode|wc -l
6042
```

2.显示指定类型的信息：

通常我只想查看某类型，比如CPU，内存或者磁盘的信息而不是全部的。这可以使用\-t(\–type TYPE)来指定信息类型

```
# dmidecode -t bios
# dmidecode -t bios, processor (这种方式不可以用，必须用下面的数字的方式)
# dmidecode -t 0,4 (显示bios和processor)
```

## 2.1 支持的type

可以在man dmidecode里面看到：
文本参数支持：
bios, system, baseboard, chassis, processor, memory, cache, connector, slot
数字参数支持很多：（见附录）
4.通过关键字查看信息：
比如只想查看序列号，可以使用:

```
# dmidecode -s system-serial-number
```

-s (–string keyword)支持的keyword包括：


bios-vendor,bios-version, bios-release-date,

system-manufacturer, system-product-name, system-version, system-serial-number,

baseboard-manu-facturer,baseboard-product-name, baseboard-version, baseboard-serial-number, baseboard-asset-tag,

chassis-manufacturer, chas-sis-version, chassis-serial-number, chassis-asset-tag,

processor-manufacturer, processor-version.



下面是基本输出信息

```
[root@localhost ~]# dmidecode
# dmidecode 3.1
Getting SMBIOS data from sysfs.
SMBIOS 3.0.0 present.
Table at 0x9AE83000.

Handle 0x0000, DMI type 0, 24 bytes
BIOS Information
	Vendor: American Megatrends Inc.
	Version: 1002
	Release Date: 07/02/2018
	Address: 0xF0000
	Runtime Size: 64 kB
	ROM Size: 16 MB
	Characteristics:
		PCI is supported
		APM is supported
		BIOS is upgradeable
		BIOS shadowing is allowed
		Boot from CD is supported
		Selectable boot is supported
		BIOS ROM is socketed
		EDD is supported
		5.25"/1.2 MB floppy services are supported (int 13h)
		3.5"/720 kB floppy services are supported (int 13h)
		3.5"/2.88 MB floppy services are supported (int 13h)
		Print screen service is supported (int 5h)
		8042 keyboard services are supported (int 9h)
		Serial services are supported (int 14h)
		Printer services are supported (int 17h)
		ACPI is supported
		USB legacy is supported
		BIOS boot specification is supported
		Targeted content distribution is supported
		UEFI is supported
	BIOS Revision: 5.12
```

# 2 如何获取DMI类型

DMI标识给我们系统的特定的硬件信息。 的dmidecode与选项'-t'或' 型 '和'ID'将为我们提供确切的infromation。 

ID 6会给我们内存模块的信息。