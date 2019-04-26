
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

AvailabilityZoneFilter - 根据可用区域进行过滤（Availability Zone元数据）。

ComputeCapabilitiesFilter - 根据计算能力进行过滤，通过请求创建虚拟机时指定的参数与主机的属性和状态进行匹配来确定是否通过，可用的操作符如下：

# 参考 

- https://my.oschina.net/LastRitter/blog/1649954

