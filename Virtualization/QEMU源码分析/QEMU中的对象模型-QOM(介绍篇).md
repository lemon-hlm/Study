
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
* [2 为什么QEMU中要实现对象模型](#2-为什么qemu中要实现对象模型)
	* [2.1 各种架构CPU的模拟和实现](#21-各种架构cpu的模拟和实现)
	* [2.2 模拟device与bus的关系](#22-模拟device与bus的关系)
* [3 QOM模型的数据结构](#3-qom模型的数据结构)

<!-- /code_chunk_output -->

# 1 概述

QEMU提供了一套面向对象编程的模型——QOM，即QEMU Object Module，几乎所有的设备如CPU、内存、总线等都是利用这一面向对象的模型来实现的。QOM模型的实现代码位于**qom/文件夹**下的文件中。

# 2 为什么QEMU中要实现对象模型

## 2.1 各种架构CPU的模拟和实现

QEMU中要实现对各种CPU架构的模拟，而且对于一种架构的CPU，比如X86\_64架构的CPU，由于包含的特性不同，也会有不同的CPU模型。

任何CPU中都有CPU通用的属性，同时也包含各自特有的属性。

为了便于模拟这些CPU模型，面向对象的编程模型是必不可少的。

## 2.2 模拟device与bus的关系

在主板上，**一个device**会通过**bus**与**其他的device**相连接，**一个device**上可以通过**不同的bus端口**连接到**其他的device**，而其他的device也可以进一步通过bus与其他的设备连接，同时一个bus上也可以连接多个device，这种**device连bus**、**bus连device的关系**，qemu是需要模拟出来的。

为了方便模拟设备的这种特性，面向对象的编程模型也是必不可少的。

# 3 QOM模型的数据结构

这些数据结构中TypeImpl定义在qom/object.c中，ObjectClass、Object、TypeInfo定义在include/qom/object.h中。 

在include/qom/object.h的注释中，对它们的每个字段都有比较明确的说明，并且说明了QOM模型的用法。 

