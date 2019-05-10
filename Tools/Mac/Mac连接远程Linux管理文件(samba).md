
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 需求背景](#1-需求背景)
* [2 samba简介](#2-samba简介)
* [3 Linux配置](#3-linux配置)
	* [3.1 安装samba](#31-安装samba)
	* [3.2 共享文件夹](#32-共享文件夹)

<!-- /code_chunk_output -->

# 1 需求背景

最近需要在Mac上远程连接一台Linux服务器，管理一些文件。不仅需要进行常规的本地文件操作，还需要上传、下载、编辑。

虽然有一些付费或免费的App，也可以完成类似工作。但其实Mac OS X自带的Finder就可以搞定了！

# 2 samba简介

samba，是一个基于GPL协议的自由软件。它重新实现了SMB/CIFS协议，可以在各个平台共享文件和打印机。

# 3 Linux配置

## 3.1 安装samba

```
yum install samba
```

## 3.2 共享文件夹

先创建一个需要共享的文件夹，这里用shared\_directory。如果已经有，直接执行chmod改变它的权限。

```
mkdir /home/USER_NAME/shared_directory
sudo chmod 777 /home/USER_NAME/shared_directory
```

USER\_NAME就是你的用户名。

