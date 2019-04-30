
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 概述

RedhatEnterprise Linux7 已经默认使用firewalld 作为防火墙，其使用方式已经变化。基于 iptables 的防火墙被默认不启动，但仍然可以继续使用。

RHEL7 中有几种防火墙共存：firewalld、iptables、ebtables等，默认使用 firewalld 作为防火墙，管理工具是firewall-cmd。使用 firewalld 来管理netfilter,不过底层调用的命令仍然是 iptables等。

因为这几种 daemon 是冲突的，所以建议禁用其他几种服务。

查看防火墙几种服务的运行状态：

# 参考

https://cloud.tencent.com/developer/article/1152579