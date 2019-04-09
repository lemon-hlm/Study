https://docs.nvidia.com/grid/latest/grid-vgpu-user-guide/index.html

# 1 基本介绍

## 1.1 NVIDIA vGPU Software使用方式

NVIDIA vGPU有三种使用方式

- NVIDIA vGPU: 一个物理GPU可以被多个虚拟机使用
- GPU Pass\-Through: 一个物理GPU被透传给一个虚拟机使用
- 裸金属部署(Bare\-Metal Deployment): 

这里介绍vGPU方式.

## 1.2 架构

NVIDIA vGPU系统架构:

![config](./images/1.png)

Hypervisor安装NVIDIA Virtual GPU Manager管理物理GPU, 从而物理GPU能够支持多个vGPU, 这些vGPU可以被虚拟机直接使用.

虚拟机使用NVIDIA Driver驱动操作vGPU.

NVIDIA vGPU内部架构:

![config](./images/2.png)

## 1.3 支持的GPU

NVIDIA vGPU作为licensed产品在Tesla GPU上可用.

要求的平台和支持的GPU, 见[NVIDIA Virtual GPU Software Documentation](https://docs.nvidia.com/grid/latest/)

### 1.3.1 Virtual GPU Types

每个物理GPU能够支持几种不同类型的Virtual GPU. Virtual GPU类型有一定数量的frame buffer, 一定数量的display heads以及最大数目的resolution. 

它们根据目标工作负载的不同类别分为不同的系列. 每个系列都由vGPU类型名称的最后一个字母标识.

- Q-series virtual GPU types面向设计人员和高级用户.
- B-series virtual GPU types面向高级用户.
- A-series virtual GPU types面向虚拟应用用户.

vGPU类型名称中的板类型后面的数字表示分配给该类型的vGPU的帧缓冲区的数量。例如，在特斯拉M60板上为M60\-2Q类型的vGPU分配2048兆字节的帧缓冲器。

由于资源要求不同，可以在物理GPU上同时创建的最大vGPU数量因vGPU类型而异。例如，Tesla M60主板可以在其两个物理GPU的每一个上支持最多4个M60-2Q vGPU，总共8个vGPU，但只有2个M60-4Q vGPU，总共4个vGPU。

注: NVIDIA vGPU在所有GPU板上都是许可证产品。需要软件许可证才能启用客户机中的vGPU的所有功能。所需的许可证类型取决于vGPU类型。

- Q-series virtual GPU types需要Quadro vDWS许可证。
- B-series virtual GPU types需要GRID Virtual PC许可证，但也可以与Quadro vDWS许可证一起使用。
- A-series virtual GPU types需要GRID虚拟应用程序许可证。

以Tesla V100 SXM2 32GB为例

![config](./images/3.png)

## 1.4 Guest VM支持

NVIDIA vGPU支持Windows和Linux虚拟机. 支持的vGPU类型取决于虚拟机的操作系统.

### 1.4.1 Windows虚拟机

支持所有的NVIDIA vGPU类型.

### 1.4.2 Linux虚拟机

64位Linux只支持Q\-series和B\-series系列NVIDIA vGPU

## 1.5 NVIDIA vGPU软件功能

NVIDIA vGPU软件包括Quadro vDWS, GRID Virtual PC, 和 GRID Virtual Applications.

# 2 安装和配置NVIDIA Virtual GPU Manager

根据Hypervisor的不同而不同. 这步完成后, 可以给虚拟机安装驱动并能license任何NVIDIA vGPU

## 2.1 准备工作

开始之前, 确保下面条件已经满足:

- 已经建好server platform, 用来托管Hypervisor和支持NVIDIA vGPU软件的NVIDIA GPU
- 在server platform, 已经安装了
- 已经下载好了适用于Hypervisor的NVIDIA vGPU软件包, 包含:
    - 针对hypervisor的NVIDIA Virtual GPU Manager管理软件
    - 针对虚拟机的NVIDIA vGPU software graphics drivers驱动
- 根据软件厂商文档安装以下软件:
    - Hypervisor, 比如KVM
    - 管理Hypervisor的软件, 比如VMware vCenter Server
    - 

## 2.2 给KVM安装NVIDIA Virtual GPU Manager

准备工作:

- 编译工具: gcc
- kernel headers: yum install kernel-headers -y
- 将相应版本内核源码放置到/usr/src/kernels/{uname -r}/
- 驱动安装文件NVIDIA-Linux-x86_64-390.113-vgpu-kvm.run

安装

```
# sh ./NVIDIA-Linux-x86_64-390.113-vgpu-kvm.run
# systemctl reboot
```

注: 如果内核源码目录不在/usr/src/kernels/{uname -r}/, 可通过--kernel-source-path=指定

验证

```
# lsmod | grep vfio
nvidia_vgpu_vfio       27099  0
nvidia              12316924  1 nvidia_vgpu_vfio
vfio_mdev              12841  0
mdev                   20414  2 vfio_mdev,nvidia_vgpu_vfio
vfio_iommu_type1       22342  0
vfio                   32331  3 vfio_mdev,nvidia_vgpu_vfio,vfio_iommu_type1
```

```
# nvidia-smi 
Tue Apr  9 14:26:40 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.113                Driver Version: 390.113                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  On   | 00000000:1A:00.0 Off |                  Off |
| N/A   33C    P0    26W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla V100-SXM2...  On   | 00000000:1B:00.0 Off |                  Off |
| N/A   37C    P0    26W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  Tesla V100-SXM2...  On   | 00000000:3D:00.0 Off |                  Off |
| N/A   34C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  Tesla V100-SXM2...  On   | 00000000:3E:00.0 Off |                  Off |
| N/A   34C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   4  Tesla V100-SXM2...  On   | 00000000:88:00.0 Off |                  Off |
| N/A   33C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   5  Tesla V100-SXM2...  On   | 00000000:89:00.0 Off |                  Off |
| N/A   34C    P0    27W / 300W |   8224MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   6  Tesla V100-SXM2...  On   | 00000000:B2:00.0 Off |                  Off |
| N/A   35C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   7  Tesla V100-SXM2...  On   | 00000000:B3:00.0 Off |                  Off |
| N/A   33C    P0    24W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## 2.3 确定vGPU type

确保想要使用vGPU的虚拟机关闭

查看物理GPU的BDF

```
# lspci | grep NVIDIA
1a:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
1b:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
3d:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
3e:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
88:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
89:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
b2:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
b3:00.0 3D controller: NVIDIA Corporation Device 1db5 (rev a1)
```

确定可用的vGPU类型对应的调解设备类型(mediated device type, mdev\_type)

```
# # ls /sys/class/mdev_bus/*/mdev_supported_types
/sys/class/mdev_bus/0000:1a:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:1b:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:3d:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:3e:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:88:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:89:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:b2:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

/sys/class/mdev_bus/0000:b3:00.0/mdev_supported_types:
nvidia-194  nvidia-196  nvidia-198  nvidia-200  nvidia-202  nvidia-204  nvidia-206  nvidia-219
nvidia-195  nvidia-197  nvidia-199  nvidia-201  nvidia-203  nvidia-205  nvidia-207

# cat /sys/class/mdev_bus/0000\:1a\:00.0/mdev_supported_types/nvidia-194/name
GRID V100DX-1Q

# cat /sys/class/mdev_bus/0000\:1a\:00.0/mdev_supported_types/nvidia-194/available_instances
32
```

其中, "GRID V100DX-1Q"是vGPU type的名字, "nvidia\-194"是"mdev\_type"标识符

所以, 对于"GRID V100DX-1Q" vGPU类型, mdev\_type标识符是"nvidia\-194"

libvirt用的就是这个mdev\_type标识符

# 3 使用vGPU

修改nova配置文件, "/etc/kolla/nova-compute/nova.conf", 添加

```
[devices]
enabled_vgpu_types = nvidia-194
```

然后重启nova\-compute服务

## 3.1 创建flavor

创建一个需要1个vGPU的flavor

```
# openstack flavor create --ram 2048 --disk 20 --vcpus 2 vgpu_1

# openstack flavor set vgpu_1 --property "resources:VGPU=1"
```

## 3.2 创建虚拟机

在"实例类型"选择"vgpu_1"

## 3.3 确认vGPU已经创建

查看虚拟机的信息

```
# vim /run/libvirt/qemu/instance-0000031d.xml

...
      <hostdev mode='subsystem' type='mdev' managed='no' model='vfio-pci' display='off'>
        <source>
          <address uuid='d878445d-df99-4510-b467-c67a4c1a7c34'/>
        </source>
        <alias name='hostdev0'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
      </hostdev>
...
```

查看系统的mdev设备

```
# ll /sys/bus/mdev/devices/
总用量 0
lrwxrwxrwx. 1 root root 0 4月   9 15:24 d878445d-df99-4510-b467-c67a4c1a7c34 -> ../../../devices/pci0000:85/0000:85:00.0/0000:86:00.0/0000:87:08.0/0000:89:00.0/d878445d-df99-4510-b467-c67a4c1a7c34
```

查看

```
[root@SH-IDC1-10-5-8-38 sunpeng1]# nvidia-smi
Tue Apr  9 15:58:54 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.113                Driver Version: 390.113                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  On   | 00000000:1A:00.0 Off |                  Off |
| N/A   33C    P0    26W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla V100-SXM2...  On   | 00000000:1B:00.0 Off |                  Off |
| N/A   37C    P0    26W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  Tesla V100-SXM2...  On   | 00000000:3D:00.0 Off |                  Off |
| N/A   34C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  Tesla V100-SXM2...  On   | 00000000:3E:00.0 Off |                  Off |
| N/A   33C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   4  Tesla V100-SXM2...  On   | 00000000:88:00.0 Off |                  Off |
| N/A   33C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   5  Tesla V100-SXM2...  On   | 00000000:89:00.0 Off |                  Off |
| N/A   34C    P0    27W / 300W |  16400MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   6  Tesla V100-SXM2...  On   | 00000000:B2:00.0 Off |                  Off |
| N/A   35C    P0    25W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   7  Tesla V100-SXM2...  On   | 00000000:B3:00.0 Off |                  Off |
| N/A   33C    P0    24W / 300W |     40MiB / 32767MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    5     25651    C+G   vgpu                                        8176MiB |
+-----------------------------------------------------------------------------+
```

# 4 给虚拟机安装驱动

准备工作

- 编译工具链: gcc
- kernel headers: yum install kernel-headers -y
- 将相应版本内核源码放置到/usr/src/kernels/{uname -r}/
- 类似于NVIDIA-Linux-x86_64-390.115-grid.run驱动安装文件

安装驱动

```
sudo sh ./NVIDIA-Linux-x86_64-390.115-grid.run \
–kernel-source-path=/usr/src/kernels/3.10.0-693.el7.x86_64
```

安装过程中

![](./images/2019-04-09-16-07-23.png)



# 3 License这个vGPU

