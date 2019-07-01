
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [ 1 获取信息](#1-获取信息)
- [ 2 下载内核源码](#2-下载内核源码)

<!-- /code_chunk_output -->

# 1 获取信息

获取centos版本

```
[root@gerrylee ~]# cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
```

获取内核版本

```
[root@gerrylee ~]# uname -r
3.10.0-957.21.3.el7.x86_64
[root@gerrylee ~]# uname -a
Linux gerrylee 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

# 2 下载内核源码

源代码的官网：http://vault.centos.org/

进入官网后，依次是进入 7.6.1810/，进入os/，进入Source/，进入SPackages/，找到kernel-3.10.0-957.el7.src.rpm，下载就行了。



参考

https://blog.csdn.net/derkampf/article/details/71189105