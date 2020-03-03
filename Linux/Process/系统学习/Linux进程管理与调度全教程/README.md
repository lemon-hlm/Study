
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 项目链接](#1-项目链接)
* [2 进程的描述](#2-进程的描述)
* [3 进程的创建](#3-进程的创建)
* [4 进程的加载与运行](#4-进程的加载与运行)
* [5 进程的退出](#5-进程的退出)
* [6 进程的调度](#6-进程的调度)
* [7 调度普通进程-完全公平调度器CFS](#7-调度普通进程-完全公平调度器cfs)

<!-- /code_chunk_output -->

# 1 项目链接

| 项目 | 描述 |
|:-------:|:-------:|
| [KernelInKernel](https://github.com/gatieme/KernelInKernel) | 一个运行在linux上的小巧内核, 修改了linux-kernel的start_kernel以启动我们自己的内核, 基于[jserv/kernel-in-kernel](https://github.com/jserv/kernel-in-kernel)(基于linux-4.1.0)和[mengning/mykernel](https://github.com/mengning/mykernel)(基于linux-3.9.4), 适合学习和研究调度算法 |
| [Linux进程管理与调度](http://blog.csdn.net/gatieme/article/category/6225543) | CSDN博客--Linux进程管理与调度 |
| [LDD-LinuxDeviceDrivers](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process) | 与CSDN博客同步更新, 但是除了包含博客的内容, 还包含了一些以驱动方式实现的实验代码 |

# 2 进程的描述

| CSDN | GitHub |
| ------------- |:-------------:|
| [Linux进程描述符task_struct结构体详解--Linux进程的管理与调度（一）](http://blog.csdn.net/gatieme/article/details/51383272)| [study/kernel/01-process/01-task/01-task_struct](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/01-task/01-task_struct) |
|[ Linux的命名空间详解--Linux进程的管理与调度（二）](http://blog.csdn.net/gatieme/article/details/51383322) | [study/kernel/01-process/01-task/02-namespace](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/01-task/02-namespace) |
|[Linux进程ID号--Linux进程的管理与调度（三）](http://blog.csdn.net/gatieme/article/details/51383377) | [study/kernel/01-process/01-task/03-pid](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/01-task/03-pid)|

# 3 进程的创建

| CSDN | GitHub |
| ------------- |:-------------:|
| [Linux下的进程类别（内核线程、轻量级进程和用户进程）以及其创建方式--Linux进程的管理与调度（四）    ](http://blog.csdn.net/gatieme/article/details/51482122) | [study/kernel/01-process/02-create/01-duplicate](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/02-create/01-duplicate)|
| [Linux下0号进程的前世(init_task进程)今生(idle进程)----Linux进程的管理与调度（五）](http://blog.csdn.net/gatieme/article/details/51484562) | [study/kernel/01-process/02-create/02-idel](http://blog.csdn.net/gatieme/article/details/51484562) |
| [Linux下1号进程的前世(kernel_init)今生(init进程)----Linux进程的管理与调度（六）](http://blog.csdn.net/gatieme/article/details/51532804) | [study/kernel/01-process/02-create/03-init](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/02-create/03-init)|
| [Linux下2号进程的kthreadd--Linux进程的管理与调度（七）](http://blog.csdn.net/gatieme/article/details/51566690) | [study/kernel/01-process/02-create/04-kthreadd](http://blog.csdn.net/gatieme/article/details/51566690) |
| [Linux下进程的创建过程分析(_do_fork/do_fork详解)--Linux进程的管理与调度（八）](http://blog.csdn.net/gatieme/article/details/51569932)| [study/kernel/01-process/02-create/05-do_fork](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/02-create/05-do_fork) |
| [Linux进程内核栈与thread_info结构详解--Linux进程的管理与调度（九）](http://blog.csdn.net/gatieme/article/details/51577479) | [study/kernel/01-process/02-create/06-thread_info](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/02-create/06-thread_info) |
| [Linux内核线程kernel thread详解--Linux进程的管理与调度（十）](http://blog.csdn.net/gatieme/article/details/51589205) | [study/kernel/01-process/02-create/07-kernel_thead](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/02-create/07-kernel_thead)|

# 4 进程的加载与运行

| CSDN | GitHub |
| ------------- |:-------------:|
| [Linux进程启动过程分析do_execve(可执行程序的加载和运行)---Linux进程的管理与调度（十一）](http://blog.csdn.net/gatieme/article/details/51594439) | [study/kernel/01-process/03-execute/01-do_execve](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/03-execute/01-do_execve) |
| [LinuxELF文件格式详解--Linux进程的管理与调度（十二）](http://blog.csdn.net/gatieme/article/details/51615799) | [study/kernel/01-process/03-execute/02-elf](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/03-execute/02-elf)|
| [ELF文件的加载过程(load_elf_binary函数详解)--Linux进程的管理与调度（十三）](http://blog.csdn.net/gatieme/article/details/51628257) |  [study/kernel/01-process/03-execute/03-load_elf_binary](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/03-execute/03-load_elf_binary) |

# 5 进程的退出

| CSDN | GitHub |
| ------------- |:-------------:|
| [Linux进程退出详解(do_exit)--Linux进程的管理与调度(十四)）](http://blog.csdn.net/gatieme/article/details/51638706) | [study/kernel/01-process/04-exit/01-do_exit](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/04-exit/01-do_exit) |

# 6 进程的调度

| CSDN | GitHub |
| ------------- |:-------------:|
| [Linux进程调度器概述--Linux进程的管理与调度(十五）](http://blog.csdn.net/gatieme/article/details/51699889) | [study/kernel/01-process/05-schedule/01-introduction](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/01-introduction) |
| [Linux进程调度策略的发展和演变--Linux进程的管理与调度(十六）](http://blog.csdn.net/gatieme/article/details/51701149)| [study/kernel/01-process/05-schedule/02-develop](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/02-develop) |
| [Linux进程调度器的设计--Linux进程的管理与调度(十七）](http://blog.csdn.net/gatieme/article/details/51702662) | [study/kernel/01-process/05-schedule/03-design](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design) |
| [Linux核心调度器之周期性调度器scheduler_tick--Linux进程的管理与调度(十八）](http://blog.csdn.net/gatieme/article/details/51872561) | [study/kernel/01-process/05-schedule/03-design/02-periodic_scheduler](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design/02-periodic_scheduler) |
| [Linux进程核心调度器之主调度器--Linux进程的管理与调度(十九）](http://blog.csdn.net/gatieme/article/details/51872594) | [study/kernel/01-process/05-schedule/03-design/03-main_scheduler](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design/03-main_scheduler) |
| [Linux用户抢占和内核抢占详解(概念, 实现和触发时机)--Linux进程的管理与调度(二十）](http://blog.csdn.net/gatieme/article/details/51872618) | [study/kernel/01-process/05-schedule/03-design/04-preempt](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design/04-preempt) |
| [Linux进程上下文切换过程context_switch详解--Linux进程的管理与调度(二十一）](http://blog.csdn.net/gatieme/article/details/51872659) | [study/kernel/01-process/05-schedule/03-design/05-context_switch](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design/05-context_switch) |
| [Linux进程优先级的处理--Linux进程的管理与调度(二十二)](http://blog.csdn.net/gatieme/article/details/51719208) | [study/kernel/01-process/05-schedule/03-design/06-priority](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design/06-priority) |
| [Linux唤醒抢占----Linux进程的管理与调度(二十三）](http://blog.csdn.net/gatieme/article/details/51872831) | [study/kernel/01-process/05-schedule/03-design/07-wakeup](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/03-design/07-wakeup) |

# 7 调度普通进程-完全公平调度器CFS

| CSDN | GitHub |
| ------------- |:-------------:|
| [Linux进程调度之CFS调度器概述--Linux进程的管理与调度(二十四）](http://blog.csdn.net/gatieme/article/details/52067518) |  [study/kernel/01-process/05-schedule/07-cfs/01-cfs/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/01-cfs) |
| [Linux CFS调度器之负荷权重load_weight--Linux进程的管理与调度(二十五）](http://blog.csdn.net/gatieme/article/details/52067665) | [study/kernel/01-process/05-schedule/07-cfs/02-load_weight/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/02-load_weight) |
| [Linux CFS调度器之虚拟时钟vruntime与调度延迟--Linux进程的管理与调度(二十六）](http://blog.csdn.net/gatieme/article/details/52067748) | [study/kernel/01-process/05-schedule/07-cfs/03-vruntime/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/03-vruntime) |
| [Linux CFS调度器之队列操作--Linux进程的管理与调度(二十七）](http://blog.csdn.net/gatieme/article/details/52067898) |  [study/kernel/01-process/05-schedule/07-cfs/04-queue/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/04-queue) |
| [Linux CFS调度器之pick_next_task_fair选择下一个被调度的进程--Linux进程的管理与调度(二十八）](http://blog.csdn.net/gatieme/article/details/52068016) | [study/kernel/01-process/05-schedule/07-cfs/05-pick_next/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/05-pick_next) |
| [Linux CFS调度器之task_tick_fair处理周期性调度器--Linux进程的管理与调度(二十九）](http://blog.csdn.net/gatieme/article/details/52068050) | [study/kernel/01-process/05-schedule/07-cfs/06-task_tick_fair/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/06-task_tick_fair) |
| [Linux CFS调度器之唤醒抢占--Linux进程的管理与调度(三十）](http://blog.csdn.net/gatieme/article/details/52068061) | [study/kernel/01-process/05-schedule/07-cfs/07-task_new_fair/](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/01-process/05-schedule/07-cfs/07-task_new_fair) |