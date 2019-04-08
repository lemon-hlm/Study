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

