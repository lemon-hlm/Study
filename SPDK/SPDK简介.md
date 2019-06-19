
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 简介](#1-简介)

<!-- /code_chunk_output -->

# 1 简介

**固态存储设备**正在取代**数据中心**。目前这一代的闪存存储，比起传统的磁盘设备，在**性能（performance**）、**功耗（power consumption**）和**机架密度（rack density**）上具有显著的优势。这些优势将会继续增大，使闪存存储作为下一代设备进入市场。

用户使用现在的**固态设备**，比如Intel® SSD DC P3700 Series Non\-Volatile Memory Express（NVMe）驱动，面临一个**主要的挑战**：因为**吞吐量**和**延迟性能**比传统的磁盘**好太多**，现在**总的处理时间**中，**存储软件**占用了**更大的比例**。换句话说，**存储软件栈**的**性能**和**效率**在整个存储系统中越来越重要。随着存储设备继续发展，它将面临远远超过正在使用的软件体系结构的风险（即**存储设备**受制于**相关软件的不足**而**不能发挥全部性能**），在接下来的几年中，存储设备将会继续发展到一个令人难以置信的地步。

为了帮助**存储OEM（设备代工厂**）和**ISV（独立软件开发商**）整合硬件，Inte构造了**一系列驱动**，以及一个**完善的**、**端对端**的**参考存储体系结构**，被命名为Storage Performance Development Kit（**SPDK**）。

SPDK的**目标**是通过**同时**使用**Intel的网络技术**，**处理技术**和**存储技术**来提高突出显著的**效率和性能**。通过运行**为硬件设计的软件**，SPDK已经证明很容易达到**每秒钟数百万次I/O读取**，通过使用许多处理器核心和许多NVMe驱动去存储，而**不需要额外卸载硬件**。

Intel在[BSD license](https://github.com/spdk/spdk/blob/master/LICENSE)许可协议下通过[Github](https://github.com/spdk)分发提供其全部的Linux参考架构的源代码。

博客、邮件列表和额外文档可以在[spdk.io](http://www.spdk.io/)中找到。




参考

- http://aidaiz.com/spdk/
- 原文：[《Introduction to the Storage Performance Development Kit (SPDK)》](https://software.intel.com/en-us/articles/introduction-to-the-storage-performance-development-kit-spdk)