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

```
# sh ./NVIDIA-Linux-x86_64-390.113-vgpu-kvm.run
# systemctl reboot
```

验证

