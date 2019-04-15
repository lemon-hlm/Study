
# 1 概述

kolla的配置管理主要是管理openstack service config文件；

主要实现是

```
kolla-ansible reconfigure
```

# 2 reconfigure使用

下面以1个例子来演示下

修改nova.conf,重启相关服务

在部署节点

```
### 修改rpc_response超时时间
vim ／etc/kolla/config/nova/nova.conf
rpc_response_timeout = 350

## 执行reconfigure
kolla-ansible reconfigure
```

# 3 reconfigure 代码流程

kolla-ansible 的核心代码在ansible实现的

先介绍下ansible role是什么？如何使用?

先看看下面nova role的代码目录

```
[root@control01 ansible]# tree -d
.
├── action_plugins
├── group_vars
├── inventory
├── library
└── roles
    ├── nova
    │   ├── defaults
    │   ├── handlers
    │   ├── meta
    │   ├── tasks
    │   └── templates
    .  
    .
    .
```

ansible role是什么？

Ansible Role 是一种分类 & 重用的概念，透过将 vars, tasks, files, templates, handler … 等等根据不同的目的(例如：nova、glance、cinder)，规划后至于独立目录中，后续便可以利用 include 的概念來使用。

若同样是 include 的概念，那 role 跟 include 之间不一样的地方又是在哪里呢?

答案是：role 的 include 机制是自动的!

我們只要事前將 role 的 vars / tasks / files / handler …. 等等事先定义好按照特定的结构(下面會提到)放好，Ansible 就會很聰明的幫我們 include 完成，不需要再自己一个一个指定 include。

透过这样的方式，管理者可以透过設定 role 的方式將所需要安裝設定的功能分门别类，拆成细项來管理并编写相对应的 script，让原本可能很庞大的设定工作可以细分成多个不同的部分來分別设定，不仅仅可以让自己重复利用特定的设定，也可以共享給其他人一同使用。

要设计一個 role，必須先知道怎麼將 files / templates / tasks / handlers / vars …. 等等拆开设定


# 参考

http://zhubingbing.cn/2017/07/09/openstack/kolla-reconfigure/

