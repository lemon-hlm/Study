
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 查看系统信息](#1-查看系统信息)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 查看系统信息

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

```


# 参考

- 该目录下: <获取CPU信息的命令>
- https://www.howtoing.com/tag/linux-tricks/