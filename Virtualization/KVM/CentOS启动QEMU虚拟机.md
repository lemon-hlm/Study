https://www.linuxidc.com/Linux/2013-05/84981.htm

在CentOS下安装KVM虚拟机，先去官网找安装的步骤，参考《[Getting KVM to run on your machine](https://www.linuxidc.com/Linux/2013-05/84981p2.htm)》

根据这篇文章，安装kvm虚拟机并运行，只需要以下三个步骤：

1. /usr/local/kvm/bin/qemu-img create -f qcow2 vdisk.img 10G 
2. /usr/local/kvm/bin/qemu-system-x86_64 -hda vdisk.img -cdrom /path/to/boot-media.iso -boot d  -m 384
3. /usr/local/kvm/bin/qemu-system-x86_64 vdisk.img -m 384

很多人在执行第1个步骤的时候，都会很顺利，不会遇到问题。大多数情况下，都会卡在第2个步骤上。在执行第2个步骤的时候，遇到的第一个问题是找不到qemu-system-x86_64命令；第二个问题就是看到"VNC server running on `::1:5900'“这个提示，google了半天也不行。

首先来说第一个问题，qemu-system-x86_64是在安装qemu（注意不是qemu-kvm）时生成的命令，而CentOS下默认安装的是qemu-kvm包，对应的命令是qemu-kvm。这个信息在上面提到的文章中也会说明，但是接着遇到的问题就是找不到qemu-kvm这个命令。qemu-kvm这个命令在/usr/libexec/目录下。对红帽系列系统比较熟的话，很容易找到qemu-kvm这个命令所在的目录，只需要通过查看rpm -ql qemu-kvm的输出即可，以后遇到类似的问题，也可以通过rpm -ql命令找到。

如果你是在桌面环境下的话，直接执行命令"vncviewer :5900“就可以继续安装过程，如果你在远程ssh连接的shell执行vncviewer命令的话，会报下面的错误：

TigerVNC Viewer for X version 1.1.0 - built Apr 29 2013 11:33:36
Copyright (C) 1999-2011 TigerVNC Team and many others (see README.txt)
See http://www.tigervnc.org for information on TigerVNC.
vncviewer: unable to open display ""

因为vncviewer需要在桌面环境下执行。

另一种方式就是在执行qemu-kvm命令的时候，加上“-vnc :0”这样就不会有这个提示了，你可以利用tightvnc这样的工具来连接到5900端口继续安装过程，这种情况的话系统不需要图形界面就可以了。

当然还有些人可能使用的方式中要在服务器段配置vncserver，这种情况的下，CentOS需要安装的rpm包为tigervnc和tigervnc-server，其中vncviewer这个命令就位于tigervnc包中。具体的安装过程参见下面两篇文章：

http://wiki.centos.org/HowTos/VNC-Server#head-76401321dae4d80916a7fd7e710272a9b85c9485

https://www.linuxidc.com/Linux/2013-04/82510.htm

在启动vncserver服务的时候，你可能遇到下面的问题：

WARNING: The first attempt to start Xvnc failed, possibly because the font
catalog is not properly configured.  Attempting to determine an appropriate
font path for this system and restart Xvnc using that font path ...
Could not start Xvnc.

/usr/bin/Xvnc: symbol lookup error: /usr/bin/Xvnc: undefined symbol: pixman\_composite\_trapezoids  
/usr/bin/Xvnc: symbol lookup error: /usr/bin/Xvnc: undefined symbol: pixman\_composite\_trapezoids

解决这个问题，只需要执行下面的命令即可：

yum install pixman pixman-devel libXfont

远程连接vncserver的工具我用的是tightvnc，这个工具是免费的，非常好用。