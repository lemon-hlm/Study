
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
	* [1.1 标准过滤器](#11-标准过滤器)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 概述

在创建一个新虚拟机实例时，Nova Scheduler通过配置好的Filter Scheduler对所有计算节点进行过滤（filtering）和称重（weighting），最后根据称重高低和用户请求节点个数返回可用主机列表。如果失败，则表明没有可用的主机。

## 1.1 标准过滤器

![](./images/2019-04-26-16-35-15.png)

- AllHostsFilter - 不进行过滤，所有可见的主机都会通过。

- ImagePropertiesFilter - 根据镜像元数据进行过滤。

- AvailabilityZoneFilter - 根据可用区域进行过滤（Availability Zone元数据）。

- ComputeCapabilitiesFilter - 根据计算能力进行过滤，通过请求创建虚拟机时指定的参数与主机的属性和状态进行匹配来确定是否通过，可用的操作符如下：

    * = (equal to or greater than as a number; same as vcpus case)
    * == (equal to as a number)
    * != (not equal to as a number)
    * \>= (greater than or equal to as a number)
    * \<= (less than or equal to as a number)
    * s== (equal to as a string)
    * s!= (not equal to as a string)
    * s\>= (greater than or equal to as a string)
    * s\> (greater than as a string)
    * s\<= (less than or equal to as a string)
    * s\< (less than as a string)
    * \<in\> (substring)
    * \<all\-in\> (all elements contained in collection)
    * \<or\> (find one of these)

Examples are: 

```
">= 5", "s== 2.1.0", "<in> gcc", "<all-in> aes mmx", and "<or> fpu <or> gpu"
```

部分可用的属性：

* free_ram_mb (compared with a number, values like ">= 4096")
* free_disk_mb (compared with a number, values like ">= 10240")
* host (compared with a string, values like: "<in> compute","s== compute_01")
* hypervisor_type (compared with a string, values like: "s== QEMU", "s== powervm")
* hypervisor_version (compared with a number, values like : ">= 1005003", "== 2000000")
* num_instances (compared with a number, values like: "<= 10")
* num_io_ops (compared with a number, values like: "<= 5")
* vcpus_total (compared with a number, values like: "= 48", ">=24")
* vcpus_used (compared with a number, values like: "= 0", "<= 10")



# 参考 

- https://my.oschina.net/LastRitter/blog/1649954

