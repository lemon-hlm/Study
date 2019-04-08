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

每个物理GPU能够支持几种不同类型的