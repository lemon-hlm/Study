
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 FSB(Front Side Bus)](#1-fsbfront-side-bus)
* [参考](#参考)

<!-- /code_chunk_output -->

CPU到现在发展已经经过了40个年头，而牙膏厂也在今年推出了8086K牙膏厂40周年cpu纪念版，用来纪念第一颗8086CPU。而CPU总线也经历了好多代的更迭，以牙膏厂为例，CPU的总线从FSB进化到QPI，而AMD则是FSB进化为HT LINK，一直到现在的GMI总线。那么今天就让我们来看看这些总线吧。

# 1 FSB(Front Side Bus)

熟悉电脑的老朋友都知道，**老的主板**是分**南北桥**的。

而**CPU**想要和**内存通信**的话，就要通过**北桥**来进行，即**CPU\-北桥\-内存**。而这个**通信总线就是FSB**。

在早期的时候，**CPU的外频**和**FSB的频率保持同步**。即**外频频率=FSB频率**，举例赛扬300A的**外频为66MHz**，那么它的**FSB频率也是66MHz**。而到了**奔腾4时代**，FSB总线速度已经无法满足CPU的带宽需求，于是牙膏厂引入了Quad Pumped Bus技术，让FSB在一个周期内可以传输四倍的数据。这就是我们最熟悉的计算方式了：FSB频率=外频频率x4，比如333MHz的外频的CPU，其FSB频率为1333MHz。大大扩展了CPU与北桥的传输速度。而FSB早期不仅仅用于CPU和北桥通信，牙膏厂早期的胶水双核也是通过FSB总线来进行数据交换的。因为牙膏厂只是简单的把两个die封装到了一个chip上，所以CPU之间想要通信必须经过北桥来进行。早期的AMD也是使用FSB总线。

![](./images/2019-04-24-10-17-28.png)



# 参考

- 本文来自知乎专栏, 链接: https://zhuanlan.zhihu.com/p/38984035?utm_source=wechat_session&utm_medium=social&utm_oi=50718148919296