
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 介绍](#1-介绍)
* [2 event\_base](#2-event_base)
	* [2.1 创建event\_base](#21-创建event_base)
	* [2.2 查看IO模型](#22-查看io模型)
	* [2.3 销毁event\_base](#23-销毁event_base)
* [参考](#参考)

<!-- /code_chunk_output -->

# 1 介绍

libevent是一个事件触发的网络库，适用于windows、linux、bsd等多种平台，内部使用select、epoll、kqueue等系统调用管理事件机制。著名分布式缓存软件memcached也是libevent based，而且libevent在使用上可以做到跨平台，而且根据libevent官方网站上公布的数据统计，似乎也有着非凡的性能。

- [官网文档](http://libevent.org/)
- [英文文档](http://www.wangafu.net/~nickm/libevent-book/)
- [中文文档](http://www.cppblog.com/mysileng/category/20374.html)

Libevent是基于事件的网络库。说的通俗点，例如我的客户端连接到服务端属于一个连接的事件，当这个事件触发的时候就会去处理。

# 2 event\_base

## 2.1 创建event\_base

event\_base是**event**（事件，后面会讲event）的一个**集合**。event\_base中存放你是监听是否就绪的event。一般情况下一个线程一个event_base，多个线程的情况下需要开多个event\_base。

event\_base主要是用来管理和实现事件的监听循环。

一般情况下直接new一个event_base就可以满足大部分需求了，如果需要配置参数的，可以参见libevent官网。

创建方法:

```c
struct event_base *event_base_new(void);
```

销毁方法:

```c
void event_base_free(struct event_base *base);
```

重新初始化:

```c
int event_reinit(struct event_base *base);
```

## 2.2 查看IO模型

IO多路复用模型中（[IO模型文章](http://blog.csdn.net/initphp/article/details/42011845)），有多种方法可以供我们选择，但是这些模型是在不同的平台下面的： select  poll  epoll  kqueue  devpoll  evport  win32

当我们创建一个event\_base的时候，libevent会**自动**为我们选择**最快的IO多路复用模型**，Linux下一般会用epoll模型。

下面这个方法主要是用来获取IO模型的名称。

```c
const char *event_base_get_method(const struct event_base *base);
```

## 2.3 销毁event\_base

```c
void event_base_free(struct event_base *base);
```




# 参考
https://blog.csdn.net/initphp/article/details/41946061

