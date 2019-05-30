
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 Systemd和Systemctl基础](#1-systemd和systemctl基础)
	* [1.1 版本](#11-版本)
	* [1.2 二进制文件和库文件](#12-二进制文件和库文件)
	* [1.3 systemd是否运行](#13-systemd是否运行)
	* [1.4 分析systemd启动进程](#14-分析systemd启动进程)
	* [1.5 分析启动时各个进程花费的时间](#15-分析启动时各个进程花费的时间)
	* [1.6 分析启动时的关键链](#16-分析启动时的关键链)
	* [1.7 列出所有可用单元](#17-列出所有可用单元)
	* [1.8 列出所有运行中单元](#18-列出所有运行中单元)
	* [1.9 列出所有失败单元](#19-列出所有失败单元)
	* [1.10 检查某个单元（如 cron.service）是否启用](#110-检查某个单元如-cronservice是否启用)
	* [1.11 检查某个单元或服务是否运行](#111-检查某个单元或服务是否运行)
* [2 使用Systemctl控制并管理服务](#2-使用systemctl控制并管理服务)
	* [2.1 列出所有服务（包括启用的和禁用的）](#21-列出所有服务包括启用的和禁用的)
	* [2.2 启动、重启、停止、重载服务以及检查服务](#22-启动-重启-停止-重载服务以及检查服务)
	* [2.3 如何激活服务并在启动时启用或禁用服务（即系统启动时自动启动服务）](#23-如何激活服务并在启动时启用或禁用服务即系统启动时自动启动服务)
	* [2.4 如何屏蔽（让它不能启动）或显示服务](#24-如何屏蔽让它不能启动或显示服务)
	* [2.5 使用systemctl命令杀死服务](#25-使用systemctl命令杀死服务)
* [3 使用Systemctl控制并管理挂载点](#3-使用systemctl控制并管理挂载点)
	* [3.1 列出所有系统挂载点](#31-列出所有系统挂载点)
	* [3.2 挂载、卸载、重新挂载、重载系统挂载点并检查系统中挂载点状态](#32-挂载-卸载-重新挂载-重载系统挂载点并检查系统中挂载点状态)
	* [3.3 在启动时激活、启用或禁用挂载点（系统启动时自动挂载）](#33-在启动时激活-启用或禁用挂载点系统启动时自动挂载)
	* [3.4 在Linux中屏蔽（让它不能启用）或可见挂载点](#34-在linux中屏蔽让它不能启用或可见挂载点)
* [4 使用Systemctl控制并管理套接口](#4-使用systemctl控制并管理套接口)
* [参考](#参考)

<!-- /code_chunk_output -->
Systemctl是一个systemd工具，主要负责控制systemd系统和服务管理器。

Systemd是一个系统管理守护进程、工具和库的集合，用于取代System V初始进程。Systemd的功能是用于集中管理和配置类UNIX系统。

在Linux生态系统中，Systemd被部署到了大多数的标准Linux发行版中，只有为数不多的几个发行版尚未部署。Systemd通常是所有其它守护进程的父进程，但并非总是如此。

# 1 Systemd和Systemctl基础

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

注意：systemd是作为父进程（PID=1）运行的。在上面带（\-e）参数的ps命令输出中，选择所有进程，（\-a）选择除会话前导外的所有进程，并使用（\-f）参数输出完整格式列表（即 \-eaf）。

## 1.4 分析systemd启动进程

```
[root@gerrylee project]# systemd-analyze
Startup finished in 12.824s (firmware) + 3.846s (loader) + 798ms (kernel) + 1.862s (initrd) + 26.028s (userspace) = 45.361s
```

## 1.5 分析启动时各个进程花费的时间

```
[root@gerrylee project]# systemd-analyze blame
         18.211s kdump.service
          6.331s NetworkManager-wait-online.service
          5.034s bolt.service
           656ms fwupd.service
           563ms systemd-udev-settle.service
           484ms dev-mapper-centos\x2droot.device
           461ms lvm2-monitor.service
...
```

## 1.6 分析启动时的关键链

```
[root@gerrylee project]# systemd-analyze critical-chain
The time after the unit is active or started is printed after the "@" character.
The time the unit takes to start is printed after the "+" character.

graphical.target @26.024s
└─multi-user.target @26.024s
  └─tuned.service @7.800s +291ms
    └─network.target @7.793s
      └─network.service @7.505s +287ms
        └─NetworkManager-wait-online.service @1.173s +6.331s
          └─NetworkManager.service @1.121s +44ms
            └─dbus.service @1.088s
              └─basic.target @1.082s
                └─paths.target @1.082s
                  └─cups.path @1.082s
                    └─sysinit.target @1.077s
                      └─systemd-update-utmp.service @1.066s +9ms
                        └─auditd.service @919ms +144ms
                          └─systemd-tmpfiles-setup.service @902ms +16ms
                            └─rhel-import-state.service @889ms +12ms
                              └─local-fs.target @887ms
                                └─run-user-42.mount @11.469s
                                  └─swap.target @800ms
                                    └─dev-mapper-centos\x2dswap.swap @784ms +15ms
                                      └─dev-mapper-centos\x2dswap.device @718ms
```

重要：Systemctl接受服务（.service），挂载点（.mount），套接口（.socket）和设备（.device）作为单元。

## 1.7 列出所有可用单元

```
[root@gerrylee project]# systemctl list-unit-files
UNIT FILE                                     STATE
proc-sys-fs-binfmt_misc.automount             static
dev-hugepages.mount                           static
dev-mqueue.mount                              static
proc-fs-nfsd.mount                            static
proc-sys-fs-binfmt_misc.mount                 static
sys-fs-fuse-connections.mount                 static
sys-kernel-config.mount                       static
sys-kernel-debug.mount                        static
tmp.mount                                     disabled
var-lib-nfs-rpc_pipefs.mount                  static
brandbot.path                                 disabled
cups.path                                     enabled
systemd-ask-password-console.path             static
systemd-ask-password-plymouth.path            static
...
```

## 1.8 列出所有运行中单元

```
[root@gerrylee project]# systemctl list-units
  UNIT                                                  LOAD   ACTIVE SUB       DESCRIPTION
  proc-sys-fs-binfmt_misc.automount                     loaded active running   Arbitrary Executable File Formats File System Automou
  sys-devices-pci0000:00-0000:00:01.0-0000:01:00.1-sound-card1.device loaded active plugged   GP106 High Definition Audio Controller
  sys-devices-pci0000:00-0000:00:17.0-ata1-host0-target0:0:0-0:0:0:0-block-sda-sda1.device loaded active plugged   ST2000DM005-2CW102
  sys-devices-pci0000:00-0000:00:17.0-ata1-host0-target0:0:0-0:0:0:0-block-sda.device loaded active plugged   ST2000DM005-2CW102
  sys-devices-pci0000:00-0000:00:17.0-ata3-host2-target2:0:0-2:0:0:0-block-sdb-sdb1.device loaded active plugged   INTEL_SSDSC2KW512G
  sys-devices-pci0000:00-0000:00:1f.6-net-enp0s31f6.device loaded active plugged   Ethernet Connection (2) I219-V
  sys-devices-platform-serial8250-tty-ttyS1.device      loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS1
  sys-devices-platform-serial8250-tty-ttyS2.device      loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS2
  sys-devices-platform-serial8250-tty-ttyS3.device      loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS3
...
```

## 1.9 列出所有失败单元

```
root@gerrylee project]# systemctl --failed
  UNIT            LOAD   ACTIVE SUB    DESCRIPTION
● ipmievd.service loaded failed failed Ipmievd Daemon

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

1 loaded units listed. Pass --all to see loaded but inactive units, too.
To show all installed unit files use 'systemctl list-unit-files'.
```

## 1.10 检查某个单元（如 cron.service）是否启用

```
[root@gerrylee project]# systemctl is-enabled crond.service
enabled
```

## 1.11 检查某个单元或服务是否运行

```
[root@gerrylee project]# systemctl is-enabled crond.service
enabled
[root@gerrylee project]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

# 2 使用Systemctl控制并管理服务

## 2.1 列出所有服务（包括启用的和禁用的）

```
[root@gerrylee project]# systemctl list-unit-files --type=service
UNIT FILE                                     STATE
abrt-oops.service                             enabled
abrt-pstoreoops.service                       disabled
abrt-vmcore.service                           enabled
abrt-xorg.service                             enabled
abrtd.service                                 enabled
accounts-daemon.service                       enabled
alsa-restore.service                          static
alsa-state.service                            static
anaconda-direct.service                       static
anaconda-nm-config.service                    static
...
```

## 2.2 启动、重启、停止、重载服务以及检查服务

```
systemctl start httpd.service
systemctl restart httpd.service
systemctl stop httpd.service
systemctl reload httpd.service
systemctl status httpd.service
```

## 2.3 如何激活服务并在启动时启用或禁用服务（即系统启动时自动启动服务）

```
systemctl is-active httpd.service
systemctl enable httpd.service
systemctl disable httpd.service
```

## 2.4 如何屏蔽（让它不能启动）或显示服务

```
# systemctl mask httpd.service
ln -s '/dev/null' '/etc/systemd/system/httpd.service'

# systemctl unmask httpd.service
rm '/etc/systemd/system/httpd.service'
```

## 2.5 使用systemctl命令杀死服务

```
systemctl kill httpd
```

# 3 使用Systemctl控制并管理挂载点

## 3.1 列出所有系统挂载点

```
systemctl list-unit-files --type=mount
```

## 3.2 挂载、卸载、重新挂载、重载系统挂载点并检查系统中挂载点状态

```
systemctl start tmp.mount
systemctl stop tmp.mount
systemctl restart tmp.mount
systemctl reload tmp.mount
systemctl status tmp.mount
```

## 3.3 在启动时激活、启用或禁用挂载点（系统启动时自动挂载）

```
systemctl is-active tmp.mount
systemctl enable tmp.mount
systemctl disable  tmp.mount
```

## 3.4 在Linux中屏蔽（让它不能启用）或可见挂载点

```
# systemctl mask tmp.mount
ln -s '/dev/null' '/etc/systemd/system/tmp.mount'

# systemctl unmask tmp.mount
rm '/etc/systemd/system/tmp.mount'
```

# 4 使用Systemctl控制并管理套接口


# 参考

https://linux.cn/article-5926-1.html