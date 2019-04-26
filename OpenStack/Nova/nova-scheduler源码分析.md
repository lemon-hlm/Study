
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 前言](#1-前言)
* [2 调度器](#2-调度器)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 前言

本篇记录了 Openstack 在创建 Instances 时，nova-scheduler 作为调度器的工作原理和代码实现。 
Openstack 中会由多个的 Instance 共享同一个 Host，而不是独占。所以就需要使用调度器这种管理规则来协调和管理 Instance 之间的资源分配。

# 2 调度器



# 参考

https://blog.csdn.net/Jmilk/article/details/52213999