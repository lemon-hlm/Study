
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 查看系统信息 \-\-\- uname](#1-查看系统信息-uname)
* [2 查看Linux系统硬件信息 \-\-\- lshw](#2-查看linux系统硬件信息-lshw)
* [3 查看Linux CPU信息 \-\-\- lscpu](#3-查看linux-cpu信息-lscpu)
* [4 收集Linux块设备信息 \-\-\- lsblk](#4-收集linux块设备信息-lsblk)
* [5 打印USB控制器信息 \-\-\- lsusb](#5-打印usb控制器信息-lsusb)
* [6 打印PCI设备信息 \-\-\- lspci](#6-打印pci设备信息-lspci)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 查看系统信息 \-\-\- uname

使用uname命令没有任何选项将打印**系统信息**或**UNAME \-s**命令将打印**系统的内核名称**。

```
[root@SH-IDC1-10-5-8-97 ~]# uname
Linux
[root@SH-IDC1-10-5-8-97 ~]# uname -s
Linux
```

要查看**网络的主机名**，请使用uname命令“\-n”开关，如图所示。

```
[root@SH-IDC1-10-5-8-97 ~]# uname -n
SH-IDC1-10-5-8-97
```

要获取有关**内核版本**，使用'\-v'开关信息。可看到这是2017年8月版本的.

```
[root@SH-IDC1-10-5-8-97 ~]# uname -v
#1 SMP Tue Aug 22 21:09:27 UTC 2017
```

**内核版本的信息(即发行编号**)，请使用“-r”开关。

```
[root@SH-IDC1-10-5-8-97 ~]# uname -r
3.10.0-693.el7.x86_64
```

**机器硬件的名称**，使用“-m”开关：

```
[root@SH-IDC1-10-5-8-97 ~]# uname -m
x86_64
```

**所有这些信息**可以通过一次如下所示运行“的uname -a”命令进行打印。

```
[root@SH-IDC1-10-5-8-97 ~]# uname -a
Linux SH-IDC1-10-5-8-97 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

# 2 查看Linux系统硬件信息 \-\-\- lshw

在这里，您可以使用**lshw工具**来收集有关您的**硬件部件**，如CPU， 磁盘 ， 内存 ，USB控制器等大量信息 lshw是一个相对较小的工具，也有一些可以在提取信息，它使用几个选项。 

通过**lshw提供的信息**聚集**不同的/proc文件**。 

注 ：请注意，以超级用户（root）或sudo用户执行的lshw命令。 

```
[root@SH-IDC1-10-5-8-97 ~]# lshw | more
sh-idc1-10-5-8-97
    description: Computer
    product: SYS-6029P-TRT (091715D9)
    vendor: Supermicro
    version: 123456789
    serial: A263370X8A19255
    width: 64 bits
    capabilities: smbios-2.8 dmi-2.8 smp vsyscall32
    configuration: boot=normal family=SMC X11 sku=091715D9 uuid=00000000-0000-0000-0000-AC1F6B4DA2D0
  *-core
       description: Motherboard
       product: X11DPi-NT
       vendor: Supermicro
       physical id: 0
       version: 1.21
       serial: NM188S036398
       slot: Default string
     *-firmware
          description: BIOS
          vendor: American Megatrends Inc.
          physical id: 0
          version: 2.1
          date: 06/29/2018
          size: 64KiB
          capacity: 32MiB
          capabilities: pci upgrade shadowing cdboot bootselect socketedrom edd int13floppy1200 int13floppy720 int13floppy2880 int5printscreen int14serial int17printer acpi usb b
iosbootspecification uefi
     *-memory
          description: System Memory
......
```

可以通过使用\-short选择打印您的**硬件信息的汇总**。

```
[root@SH-IDC1-10-5-8-97 ~]# lshw -short
H/W path          Device          Class          Description
============================================================
                                  system         SYS-6029P-TRT (091715D9)
/0                                bus            X11DPi-NT
/0/0                              memory         64KiB BIOS
/0/13                             memory         128GiB System Memory
/0/13/0                           memory         32GiB DIMM DDR4 Synchronous 2666 MHz (0.4 ns)
/0/13/1                           memory         [empty]
/0/13/2                           memory         [empty]
/0/13/3                           memory         [empty]
/0/13/4                           memory         32GiB DIMM DDR4 Synchronous 2666 MHz (0.4 ns)
/0/13/5                           memory         [empty]
/0/13/6                           memory         [empty]
/0/13/7                           memory         [empty]
/0/13/8                           memory         32GiB DIMM DDR4 Synchronous 2666 MHz (0.4 ns)
......
```

如果想生成**一个HTML文件输出**，可以使用该选项-html。

```
[root@SH-IDC1-10-5-8-97 ~]# lshw -html > /root/lshw.html
```

![](./images/2019-04-20-22-48-19.png)

# 3 查看Linux CPU信息 \-\-\- lscpu

要查看你的CPU信息，请使用lscpu命令，因为它显示你的CPU体系结构的信息，如从**sysfs**中和的/**proc/cpuinfo** CPU的数量，内核，CPU系列型号，CPU高速缓存，线程等。

```
[root@SH-IDC1-10-5-8-97 ~]# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                64
On-line CPU(s) list:   0-63
Thread(s) per core:    2
Core(s) per socket:    16
座：                 2
NUMA 节点：         2
厂商 ID：           GenuineIntel
CPU 系列：          6
型号：              85
型号名称：        Intel(R) Xeon(R) Gold 6130 CPU @ 2.10GHz
步进：              4
CPU MHz：             1100.000
CPU max MHz:           2101.0000
CPU min MHz:           1000.0000
BogoMIPS：            4200.00
虚拟化：           VT-x
L1d 缓存：          32K
L1i 缓存：          32K
L2 缓存：           1024K
L3 缓存：           22528K
NUMA 节点0 CPU：    0-15,32-47
NUMA 节点1 CPU：    16-31,48-63
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch epb cat_l3 cdp_l3 intel_pt tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm cqm mpx rdt_a avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local dtherm ida arat pln pts
```

# 4 收集Linux块设备信息 \-\-\- lsblk

**块设备是存储设备**，诸如**硬盘，闪存驱动器**等lsblk命令用于报告关于**块设备的信息**如下。

```
[root@SH-IDC1-10-5-8-97 ~]# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   500G  0 disk
├─sda1   8:1    0     1G  0 part /boot
├─sda2   8:2    0   459G  0 part /
└─sda3   8:3    0    40G  0 part [SWAP]
sdb      8:16   0   1.3T  0 disk
sdc      8:32   0 893.8G  0 disk
```

如果你想查看系统上**所有的块设备**则包括\-a选项。

```
[root@SH-IDC1-10-5-8-97 ~]# lsblk -a
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   500G  0 disk
├─sda1   8:1    0     1G  0 part /boot
├─sda2   8:2    0   459G  0 part /
└─sda3   8:3    0    40G  0 part [SWAP]
sdb      8:16   0   1.3T  0 disk
sdc      8:32   0 893.8G  0 disk
```

# 5 打印USB控制器信息 \-\-\- lsusb

所述的**lsusb**命令用于报告关于**USB控制器**和所有**连接到他们的设备**的信息。

```
[root@SH-IDC1-10-5-8-97 ~]# lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 0557:2419 ATEN International Co., Ltd
Bus 001 Device 002: ID 0557:7000 ATEN International Co., Ltd Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

可以使用\-v选项生成**每个USB设备**的**详细信息**。

```
[root@SH-IDC1-10-5-8-97 ~]# lsusb -v
```

# 6 打印PCI设备信息 \-\-\- lspci

**PCI设备**可以包括**USB接口**，**显卡**，**网卡**等lspci的工具，用于生成有关系统上的**所有PCI控制器**以及**连接到这些设备的信息**。 要打印有关PCI设备的信息，请运行以下命令。

```
[root@SH-IDC1-10-5-8-97 ~]# lspci
00:00.0 Host bridge: Intel Corporation Device 2020 (rev 04)
00:04.0 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.1 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.2 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.3 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.4 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.5 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.6 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:04.7 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
00:05.0 System peripheral: Intel Corporation Sky Lake-E MM/Vt-d Configuration Registers (rev 04)
00:05.2 System peripheral: Intel Corporation Device 2025 (rev 04)
00:05.4 PIC: Intel Corporation Device 2026 (rev 04)
00:08.0 System peripheral: Intel Corporation Sky Lake-E Ubox Registers (rev 04)
00:08.1 Performance counters: Intel Corporation Sky Lake-E Ubox Registers (rev 04)
00:08.2 System peripheral: Intel Corporation Sky Lake-E Ubox Registers (rev 04)
00:11.0 Unassigned class [ff00]: Intel Corporation Device a1ec (rev 09)
00:11.1 Unassigned class [ff00]: Intel Corporation Device a1ed (rev 09)
00:11.5 SATA controller: Intel Corporation Lewisburg SSATA Controller [AHCI mode] (rev 09)
00:14.0 USB controller: Intel Corporation Lewisburg USB 3.0 xHCI Controller (rev 09)
00:14.2 Signal processing controller: Intel Corporation Device a1b1 (rev 09)
00:16.0 Communication controller: Intel Corporation Lewisburg CSME: HECI #1 (rev 09)
00:16.1 Communication controller: Intel Corporation Lewisburg CSME: HECI #2 (rev 09)
00:16.4 Communication controller: Intel Corporation Lewisburg CSME: HECI #3 (rev 09)
00:17.0 SATA controller: Intel Corporation Lewisburg SATA Controller [AHCI mode] (rev 09)
00:1c.0 PCI bridge: Intel Corporation Lewisburg PCI Express Root Port #1 (rev f9)
00:1c.5 PCI bridge: Intel Corporation Lewisburg PCI Express Root Port #6 (rev f9)
00:1f.0 ISA bridge: Intel Corporation Lewisburg LPC Controller (rev 09)
00:1f.2 Memory controller: Intel Corporation Lewisburg PMC (rev 09)
00:1f.4 SMBus: Intel Corporation Lewisburg SMBus (rev 09)
00:1f.5 Serial bus controller [0c80]: Intel Corporation Lewisburg SPI Controller (rev 09)
02:00.0 PCI bridge: ASPEED Technology, Inc. AST1150 PCI-to-PCI Bridge (rev 04)
03:00.0 VGA compatible controller: ASPEED Technology, Inc. ASPEED Graphics Family (rev 41)
17:05.0 System peripheral: Intel Corporation Device 2034 (rev 04)
17:05.2 System peripheral: Intel Corporation Sky Lake-E RAS Configuration Registers (rev 04)
17:05.4 PIC: Intel Corporation Device 2036 (rev 04)
17:08.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:08.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:09.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0a.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0b.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0b.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0b.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0b.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0e.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:0f.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:10.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:11.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:11.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:11.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:11.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:1d.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:1d.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:1d.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:1d.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
17:1e.0 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
17:1e.1 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
17:1e.2 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
17:1e.3 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
17:1e.4 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
17:1e.5 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
17:1e.6 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
3a:00.0 PCI bridge: Intel Corporation Sky Lake-E PCI Express Root Port A (rev 04)
3a:05.0 System peripheral: Intel Corporation Device 2034 (rev 04)
3a:05.2 System peripheral: Intel Corporation Sky Lake-E RAS Configuration Registers (rev 04)
3a:05.4 PIC: Intel Corporation Device 2036 (rev 04)
3a:08.0 System peripheral: Intel Corporation Device 2066 (rev 04)
3a:09.0 System peripheral: Intel Corporation Device 2066 (rev 04)
3a:0a.0 System peripheral: Intel Corporation Device 2040 (rev 04)
3a:0a.1 System peripheral: Intel Corporation Device 2041 (rev 04)
3a:0a.2 System peripheral: Intel Corporation Device 2042 (rev 04)
3a:0a.3 System peripheral: Intel Corporation Device 2043 (rev 04)
3a:0a.4 System peripheral: Intel Corporation Device 2044 (rev 04)
3a:0a.5 System peripheral: Intel Corporation Device 2045 (rev 04)
3a:0a.6 System peripheral: Intel Corporation Device 2046 (rev 04)
3a:0a.7 System peripheral: Intel Corporation Device 2047 (rev 04)
3a:0b.0 System peripheral: Intel Corporation Device 2048 (rev 04)
3a:0b.1 System peripheral: Intel Corporation Device 2049 (rev 04)
3a:0b.2 System peripheral: Intel Corporation Device 204a (rev 04)
3a:0b.3 System peripheral: Intel Corporation Device 204b (rev 04)
3a:0c.0 System peripheral: Intel Corporation Device 2040 (rev 04)
3a:0c.1 System peripheral: Intel Corporation Device 2041 (rev 04)
3a:0c.2 System peripheral: Intel Corporation Device 2042 (rev 04)
3a:0c.3 System peripheral: Intel Corporation Device 2043 (rev 04)
3a:0c.4 System peripheral: Intel Corporation Device 2044 (rev 04)
3a:0c.5 System peripheral: Intel Corporation Device 2045 (rev 04)
3a:0c.6 System peripheral: Intel Corporation Device 2046 (rev 04)
3a:0c.7 System peripheral: Intel Corporation Device 2047 (rev 04)
3a:0d.0 System peripheral: Intel Corporation Device 2048 (rev 04)
3a:0d.1 System peripheral: Intel Corporation Device 2049 (rev 04)
3a:0d.2 System peripheral: Intel Corporation Device 204a (rev 04)
3a:0d.3 System peripheral: Intel Corporation Device 204b (rev 04)
3b:00.0 Ethernet controller: Intel Corporation Ethernet Controller 10-Gigabit X540-AT2 (rev 01)
3b:00.1 Ethernet controller: Intel Corporation Ethernet Controller 10-Gigabit X540-AT2 (rev 01)
5d:02.0 PCI bridge: Intel Corporation Sky Lake-E PCI Express Root Port C (rev 04)
5d:05.0 System peripheral: Intel Corporation Device 2034 (rev 04)
5d:05.2 System peripheral: Intel Corporation Sky Lake-E RAS Configuration Registers (rev 04)
5d:05.4 PIC: Intel Corporation Device 2036 (rev 04)
5d:0e.0 Performance counters: Intel Corporation Device 2058 (rev 04)
5d:0e.1 System peripheral: Intel Corporation Device 2059 (rev 04)
5d:0f.0 Performance counters: Intel Corporation Device 2058 (rev 04)
5d:0f.1 System peripheral: Intel Corporation Device 2059 (rev 04)
5d:10.0 Performance counters: Intel Corporation Device 2058 (rev 04)
5d:10.1 System peripheral: Intel Corporation Device 2059 (rev 04)
5d:12.0 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
5d:12.1 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
5d:12.2 System peripheral: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
5d:12.4 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
5d:12.5 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
5d:15.0 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
5d:16.0 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
5d:16.4 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
5d:17.0 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
5e:00.0 PCI bridge: Intel Corporation Device 37c0 (rev 09)
5f:03.0 PCI bridge: Intel Corporation Device 37c5 (rev 09)
60:00.0 Ethernet controller: Intel Corporation Ethernet Connection X722 for 10GBASE-T (rev 09)
60:00.1 Ethernet controller: Intel Corporation Ethernet Connection X722 for 10GBASE-T (rev 09)
80:04.0 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.1 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.2 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.3 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.4 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.5 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.6 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:04.7 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
80:05.0 System peripheral: Intel Corporation Sky Lake-E MM/Vt-d Configuration Registers (rev 04)
80:05.2 System peripheral: Intel Corporation Device 2025 (rev 04)
80:05.4 PIC: Intel Corporation Device 2026 (rev 04)
80:08.0 System peripheral: Intel Corporation Sky Lake-E Ubox Registers (rev 04)
80:08.1 Performance counters: Intel Corporation Sky Lake-E Ubox Registers (rev 04)
80:08.2 System peripheral: Intel Corporation Sky Lake-E Ubox Registers (rev 04)
85:00.0 PCI bridge: Intel Corporation Sky Lake-E PCI Express Root Port A (rev 04)
85:05.0 System peripheral: Intel Corporation Device 2034 (rev 04)
85:05.2 System peripheral: Intel Corporation Sky Lake-E RAS Configuration Registers (rev 04)
85:05.4 PIC: Intel Corporation Device 2036 (rev 04)
85:08.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:08.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:09.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0a.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0b.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0b.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0b.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0b.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0e.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:0f.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.4 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.5 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.6 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:10.7 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:11.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:11.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:11.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:11.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:1d.0 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:1d.1 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:1d.2 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:1d.3 System peripheral: Intel Corporation Sky Lake-E CHA Registers (rev 04)
85:1e.0 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
85:1e.1 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
85:1e.2 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
85:1e.3 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
85:1e.4 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
85:1e.5 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
85:1e.6 System peripheral: Intel Corporation Sky Lake-E PCU Registers (rev 04)
86:00.0 RAID bus controller: LSI Logic / Symbios Logic MegaRAID SAS-3 3108 [Invader] (rev 02)
ae:00.0 PCI bridge: Intel Corporation Sky Lake-E PCI Express Root Port A (rev 04)
ae:05.0 System peripheral: Intel Corporation Device 2034 (rev 04)
ae:05.2 System peripheral: Intel Corporation Sky Lake-E RAS Configuration Registers (rev 04)
ae:05.4 PIC: Intel Corporation Device 2036 (rev 04)
ae:08.0 System peripheral: Intel Corporation Device 2066 (rev 04)
ae:09.0 System peripheral: Intel Corporation Device 2066 (rev 04)
ae:0a.0 System peripheral: Intel Corporation Device 2040 (rev 04)
ae:0a.1 System peripheral: Intel Corporation Device 2041 (rev 04)
ae:0a.2 System peripheral: Intel Corporation Device 2042 (rev 04)
ae:0a.3 System peripheral: Intel Corporation Device 2043 (rev 04)
ae:0a.4 System peripheral: Intel Corporation Device 2044 (rev 04)
ae:0a.5 System peripheral: Intel Corporation Device 2045 (rev 04)
ae:0a.6 System peripheral: Intel Corporation Device 2046 (rev 04)
ae:0a.7 System peripheral: Intel Corporation Device 2047 (rev 04)
ae:0b.0 System peripheral: Intel Corporation Device 2048 (rev 04)
ae:0b.1 System peripheral: Intel Corporation Device 2049 (rev 04)
ae:0b.2 System peripheral: Intel Corporation Device 204a (rev 04)
ae:0b.3 System peripheral: Intel Corporation Device 204b (rev 04)
ae:0c.0 System peripheral: Intel Corporation Device 2040 (rev 04)
ae:0c.1 System peripheral: Intel Corporation Device 2041 (rev 04)
ae:0c.2 System peripheral: Intel Corporation Device 2042 (rev 04)
ae:0c.3 System peripheral: Intel Corporation Device 2043 (rev 04)
ae:0c.4 System peripheral: Intel Corporation Device 2044 (rev 04)
ae:0c.5 System peripheral: Intel Corporation Device 2045 (rev 04)
ae:0c.6 System peripheral: Intel Corporation Device 2046 (rev 04)
ae:0c.7 System peripheral: Intel Corporation Device 2047 (rev 04)
ae:0d.0 System peripheral: Intel Corporation Device 2048 (rev 04)
ae:0d.1 System peripheral: Intel Corporation Device 2049 (rev 04)
ae:0d.2 System peripheral: Intel Corporation Device 204a (rev 04)
ae:0d.3 System peripheral: Intel Corporation Device 204b (rev 04)
af:00.0 Infiniband controller: Mellanox Technologies MT28908 Family [ConnectX-6]
d7:05.0 System peripheral: Intel Corporation Device 2034 (rev 04)
d7:05.2 System peripheral: Intel Corporation Sky Lake-E RAS Configuration Registers (rev 04)
d7:05.4 PIC: Intel Corporation Device 2036 (rev 04)
d7:0e.0 Performance counters: Intel Corporation Device 2058 (rev 04)
d7:0e.1 System peripheral: Intel Corporation Device 2059 (rev 04)
d7:0f.0 Performance counters: Intel Corporation Device 2058 (rev 04)
d7:0f.1 System peripheral: Intel Corporation Device 2059 (rev 04)
d7:10.0 Performance counters: Intel Corporation Device 2058 (rev 04)
d7:10.1 System peripheral: Intel Corporation Device 2059 (rev 04)
d7:12.0 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
d7:12.1 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
d7:12.2 System peripheral: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
d7:12.4 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
d7:12.5 Performance counters: Intel Corporation Sky Lake-E M3KTI Registers (rev 04)
d7:15.0 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
d7:16.0 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
d7:16.4 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
d7:17.0 System peripheral: Intel Corporation Sky Lake-E M2PCI Registers (rev 04)
```





# 参考

- 该目录下: <获取CPU信息的命令>
- https://www.howtoing.com/tag/linux-tricks/