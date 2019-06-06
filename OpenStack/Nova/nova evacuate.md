
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 evacuate实例](#1-evacuate实例)
* [2 撤离一个实例](#2-撤离一个实例)
* [3 撤离所有实例](#3-撤离所有实例)
* [4 源码分析](#4-源码分析)

<!-- /code_chunk_output -->

# 1 evacuate实例

nova evacuate 实现当虚拟机所在宿主机出现宕机后，虚拟机可以通过evacuate将虚拟机从宕机的物理节点迁移至其它节点。 

nova evacuate其实是通过虚拟机rebuild的过程完成的，原compute节点在重新恢复后会进行虚拟机删除

如果您需要把一个实例从一个有故障或已停止运行的 compute 节点上移到同一个环境中的其它主机服务器上时，可以使用 nova evacuate 命令对实例进行撤离（evacuate）。

- 撤离的操作只有在实例的磁盘在共享存储上，或实例的磁盘是块存储卷时才有效。因为在其它情况下，磁盘无法被新 compute 节点访问。

- 实例只有在它所在的服务器停止运行的情况下才可以被撤离；如果服务器没有被关闭，evacuate 命令会运行失败。

# 2 撤离一个实例

使用以下命令撤离一个实例：

```
# nova evacuate [--password pass] [--on-shared-storage] instance_name [target_host]
```

其中：

- \-\-password pass \- 撤离实例的 admin 密码（如果指定了 \-\-on\-shared\-storage，则无法使用）。如果没有指定密码，一个随机密码会被产生，并在**撤离操作完成后被输出**。

- \-\-on\-shared\-storage \- 指定所有实例文件都在共享存储中。

- instance\_name \- 要撤离的实例名称。

- target\_host \- 实例撤离到的主机；如果您没有指定这个主机，Compute 调度会为您选择一个主机。您可以使用以下命令找到可能的主机：

```
# nova host-list | grep compute
```

例如：

```
# nova evacuate myDemoInstance Compute2_OnEL7.myDomain
```

# 3 撤离所有实例

使用以下命令在一个特定主机上撤离所有实例：

```
# nova host-evacuate instance_name [--target target_host] [--on-shared-storage] source_host
```

其中：


- \-\-target target\_host \- 实例撤离到的主机；如果您没有指定这个主机，Compute 调度会为您选择一个主机。您可以使用以下命令找到可能的主机：

```
# nova host-list | grep compute
```

- \-\-on\-shared\-storage \- 指定所有实例文件都在共享存储中。

- source\_host \- 被撤离的主机名。

例如：

```
# nova host-evacuate --target Compute2_OnEL7.localdomain myDemoHost.localdomain
```

# 4 源码分析

nova evacuate 实现当虚拟机所在宿主机出现宕机后，虚拟机可以通过evacuate将虚拟机从宕机的物理节点迁移至其它节点。 

nova evacuate其实是通过虚拟机rebuild的过程完成的，原compute节点在重新恢复后会进行虚拟机删除

入口文件nova/api/openstack/compute/contrib/evacuate.py中Controller类下的_evacuate方法


参考

- https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux_OpenStack_Platform/6/html/Administration_Guide/section-evacuation.html

- https://blog.fabian4.cn/2016/10/27/nova-evacuate/