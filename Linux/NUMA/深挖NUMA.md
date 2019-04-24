
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 相关内容](#1-相关内容)
* [2](#2)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 相关内容

- [Linux的NUMA机制](https://link.zhihu.com/?target=http%3A//www.litrin.net/2014/06/18/linux%25e7%259a%2584numa%25e6%259c%25ba%25e5%2588%25b6/)
- [NUMA对性能的影响](https://link.zhihu.com/?target=http%3A//www.litrin.net/2017/08/03/numa%25e5%25af%25b9%25e6%2580%25a7%25e8%2583%25bd%25e7%259a%2584%25e5%25bd%25b1%25e5%2593%258d/)
- [cgroup的cpuset问题](https://link.zhihu.com/?target=http%3A//www.litrin.net/2016/05/18/cgroup%25e7%259a%2584cpuset%25e9%2597%25ae%25e9%25a2%2598/)

# 2 

在若干年前，对于x86架构的计算机，那时的**内存控制器还没有整合进CPU**，所有**内存的访问**都需要通过**北桥芯片**来完成。此时的内存访问如下图所示，被称为UMA（uniform memory access, 一致性内存访问 ）。这样的访问对于软件层面来说非常容易实现：总线模型保证了所有的内存访问是一致的，不必考虑由不同内存地址之前的差异。

![](./images/2019-04-24-09-00-59.png)



# 参考

- 本文来自于知乎专栏, 链接: https://zhuanlan.zhihu.com/p/33621500?utm_source=wechat_session&utm_medium=social&utm_oi=50718148919296