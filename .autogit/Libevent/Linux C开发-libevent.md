
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 介绍](#1-介绍)
* [2 event\_base](#2-event_base)
	* [2.1 创建event\_base](#21-创建event_base)
	* [2.2 查看IO模型](#22-查看io模型)
	* [2.3 销毁event\_base](#23-销毁event_base)
	* [2.4 事件循环 event loop](#24-事件循环-event-loop)
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

## 2.4 事件循环 event loop

上面说到 event\_base是一组event的集合，我们也可以将event事件注册到这个集合中。当需要事件监听的时候，我们就需要对这个event\_base进行循环。

下面这个函数非常重要，会在内部不断的循环监听注册上来的事件。

```c
int event_base_dispatch(struct event_base *base);
```

返回值：0 表示成功退出  -1 表示存在错误信息。

还可以用这个方法：

```c
#define EVLOOP_ONCE             0x01
#define EVLOOP_NONBLOCK         0x02
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04
 
int event_base_loop(struct event_base *base, int flags);
```

event_base_loop这个方法会比event_base_dispatch这个方法更加灵活一些。

- EVLOOP\_ONCE: 阻塞直到有一个活跃的event，然后执行完活跃事件的回调就退出。

- EVLOOP\_NONBLOCK: 不阻塞，检查哪个事件准备好，调用优先级最高的那一个，然后退出。

0：如果参数填了0，则只有事件进来的时候才会调用一次事件的回调函数，比较常用

事件循环停止的情况：

1. event_base中没有事件event

2. 调用event_base_loopbreak()，那么事件循环将停止

3. 调用event_base_loopexit()，那么事件循环将停止

4. 程序错误，异常退出

两个退出的方法：

```c
// 这两个函数成功返回 0 失败返回 -1
// 指定在 tv 时间后停止事件循环
// 如果 tv == NULL 那么将无延时的停止事件循环
int event_base_loopexit(struct event_base *base,const struct timeval *tv);
// 立即停止事件循环（而不是无延时的停止）
int event_base_loopbreak(struct event_base *base);
```



# 参考
https://blog.csdn.net/initphp/article/details/41946061

