
# 1 cat命令 \- 获取CPU信息

```
# cat /proc/cpuinfo
```

# 2 lscpu命令 \- 显示CPU架构信息

命令lscpu从sysfs和/proc/cpuinfo打印CPU体系结构信息

```
# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                12
On-line CPU(s) list:   0-11
Thread(s) per core:    2
Core(s) per socket:    6
座：                 1
NUMA 节点：         1
厂商 ID：           GenuineIntel
CPU 系列：          6
型号：              158
型号名称：        Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
步进：              10
CPU MHz：             898.632
CPU max MHz:           4200.0000
CPU min MHz:           800.0000
BogoMIPS：            6384.00
虚拟化：           VT-x
L1d 缓存：          32K
L1i 缓存：          32K
L2 缓存：           256K
L3 缓存：           12288K
NUMA 节点0 CPU：    0-11
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch epb intel_pt ssbd ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 dtherm ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp spec_ctrl intel_stibp flush_l1d
```

# 3 cpuid命令 \- 显示x86 CPU

命令cpuid转储从CPUID指令收集的CPU的完整信息，并从该信息中发现x86 CPU的确切型号。

安装包

```
$ sudo apt install cpuid        #Debian/Ubuntu systems
$ sudo yum install cpuid	#RHEL/CentOS systems 
$ sudo dnf install cpuid	#Fedora 22+ 
```

命令

```
# cpuid
```

