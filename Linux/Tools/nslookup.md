
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1 nslookup作用](#1-nslookup作用)
- [2 A记录](#2-a记录)

<!-- /code_chunk_output -->

# 1 nslookup作用

nslookup用于**查询DNS的记录**，查询**域名解析是否正常**，在网络故障时用来诊断网络问题

```
[root@gerry ~]# nslookup
>
```

# 2 A记录

A（Address）记录指的是用来指定主机名或域名对应的IP记录。

在提示符\>后直接输入域名，可以查看该域名的A记录（也可以用set type=a指令设置）：

```
[root@gerry ~]# nslookup
> bilibili.com
Server:		172.16.100.2
Address:	172.16.100.2#53

Non-authoritative answer:
Name:	bilibili.com
Address: 106.75.240.122
>
```

