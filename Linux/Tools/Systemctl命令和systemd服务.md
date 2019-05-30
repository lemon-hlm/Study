
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Systemctl基础](#1-systemctl基础)
	* [1.1 版本](#11-版本)
	* [1.2 二进制文件和库文件](#12-二进制文件和库文件)
	* [1.3 systemd是否运行](#13-systemd是否运行)
* [参考](#参考)

<!-- /code_chunk_output -->
Systemctl是一个systemd工具，主要负责控制systemd系统和服务管理器。

Systemd是一个系统管理守护进程、工具和库的集合，用于取代System V初始进程。Systemd的功能是用于集中管理和配置类UNIX系统。

在Linux生态系统中，Systemd被部署到了大多数的标准Linux发行版中，只有为数不多的几个发行版尚未部署。Systemd通常是所有其它守护进程的父进程，但并非总是如此。

# 1 Systemctl基础

## 1.1 版本

首先检查你的系统中是否安装有**systemd**并确定当前安装的版本

```
[root@gerrylee ~]# systemctl --version
systemd 219
+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 -SECCOMP +BLKID +ELFUTILS +KMOD +IDN
```

## 1.2 二进制文件和库文件

检查systemd和systemctl的二进制文件和库文件的安装位置

```
[root@gerrylee ~]# whereis systemd
systemd: /usr/lib/systemd /etc/systemd /usr/share/systemd /usr/share/man/man1/systemd.1.gz
[root@gerrylee ~]# whereis systemctl
systemctl: /usr/bin/systemctl /usr/share/man/man1/systemctl.1.gz
```

## 1.3 systemd是否运行

```
[root@gerrylee ~]# ps -eaf | grep [s]ystemd
root         1     0  0 5月21 ?       00:02:54 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
root      2656     1  0 5月21 ?       00:00:04 /usr/lib/systemd/systemd-journald
root      2692     1  0 5月21 ?       00:00:00 /usr/lib/systemd/systemd-udevd
dbus      5466     1  7 5月21 ?       16:21:20 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
root      5501     1  0 5月21 ?       00:00:07 /usr/lib/systemd/systemd-logind
```

# 参考

https://linux.cn/article-5926-1.html