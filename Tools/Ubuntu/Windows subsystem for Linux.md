### 1. Windows10更新

将windows10更新到最新系统

### 2. 打开功能

“控制面板” —— “程序” —— “程序和功能” —— “启用或关闭Windows功能” —— 打开“适用于Linux的Windows子系统”

### 3. 安装ubuntu

在Windows Store中搜索并安装ubuntu

### 4. 启用root以及ssh

替换掉/etc/shadow或者修改里面内容，将root密码删掉（置空）

修改ssh服务的配置文件（一般在/etc/ssh/sshd_config），参照当前目录下文件。

在/root/.bashrc尾添加下面内容

```
service ssh status > /dev/null

if [ $? -ne 0 ]
then
    service ssh start
else
    echo "ssh service is already running"
fi
```

### 5. 修改ubuntu的软件源

ubuntu的软件源文件是/etc/apt/sources.list。

查看当前OS的版本：

```
root@Gerry:~# cat /etc/issue
Ubuntu 16.04.3 LTS \n \l

root@Gerry:~# lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.3 LTS
Release:        16.04
Codename:       xenial
```

清华镜像源配置：https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/

根据版本选择一个源。

### 6. 更新下所有软件

```
apt-get dist-upgrade
```