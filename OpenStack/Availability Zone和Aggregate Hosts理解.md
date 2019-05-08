
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 availability zone](#1-availability-zone)
* [2 Aggregate Hosts](#2-aggregate-hosts)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 availability zone

az是在**region范围内**的再次切分，只是**工程上的独立**，例如可以把一个机架上的机器划分在一个az中，划分az是为了提高**容灾性**和提供**廉价的隔离服务**。选择不同的region主要考虑哪个region靠近你的用户群体，比如用户在美国，自然会选择离美国近的region；选择不同的az是为了防止所有的instance一起挂掉，下图描述了二者之间的关系。

![](./images/2019-05-08-11-02-53.png)

catalog其实是分级的  ...，第二级的region就是上文提到的region。在这里我们可以设置不同的region和不同的service的endpoint。horizon只提取keystone中catalog的regionOne中的endpoint，所以，即使设置了多个region，horizon也体现不出来。

az在openstack中其实是nova\-scheduler来实现的，当新建虚拟机，调度器将会根据nova-compute设置的az来调度，例如在新建虚拟机的时候，用户设置了希望将虚拟机放在az-1中，那么调度器将会选择属于这个az的nova\-compute来调度，如下图所示。

![](./images/2019-05-08-11-04-17.png)

```
#nova boot --image 1fe4b52c-bda5-11e2-a40b-f23c91aec05e --flavor large --availability-zone chicago:clocktower
```

指定instance clocktower将在availability zone\-chicago被创建，至于那些compute node属于哪一个az，是在nova.conf中通过参数node\_availability\_zone=xxx来配置的。

# 2 Aggregate Hosts

 Availability zones are a customer-facing capability, host aggregates are meant to be used by administrators to separate hardware by particular properties, and are not seen by customers.

az是一个面向终端客户的概念和能力，而host aggregate是管理员用来根据硬件资源的某一属性来对硬件进行划分的功能，只对管理员可见。

其主要功能就是实现根据某一属性来划分物理机，比如按照地理位置，使用固态硬盘的机器，内存超过32G的机器，根据这些指标来构成一个host group。

nova aggregate-create joesservers chicago

Host aggregate可以用来进一步细分availability zone。

通过以上分析，问题就来了：availability zone和host aggregate都能对host machine进行划分，那么二者的区别是啥？

Availability zones are handy for allowing users to specify a particular group of servers on which they want their host to run, but beyond that they don’t do much more than serve as a bucket. In this example, using an availability zone, our users can specify that a VM should be started up in the Chicago data center.

Host aggregates, on the other hand, serve as an intelligent way for schedulers to know where to place VM’s based on some sort of characteristic. In this example, we might want to enable users to easily boot their most mission-critical VMs on servers that are administered by Joe, rather than leaving them to fate.

综上所述：az是用户可见的，用户手动的来指定vm运行在哪些host上；Host aggregate是一种更智能的方式，是调度器可见的，影响调度策略的一个表达式。

# 参考

http://blog.chinaunix.net/uid-20940095-id-3875022.html