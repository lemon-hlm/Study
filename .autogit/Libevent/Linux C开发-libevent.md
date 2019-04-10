
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 介绍](#1-介绍)
* [2 event\_base](#2-event_base)
	* [2.1 创建event\_base](#21-创建event_base)
	* [2.2 查看IO模型](#22-查看io模型)
	* [2.3 销毁event\_base](#23-销毁event_base)
	* [2.4 事件循环 event loop](#24-事件循环-event-loop)
	* [2.5 event\_base的例子](#25-event_base的例子)
* [3 event事件](#3-event事件)
	* [3.1 创建一个事件event](#31-创建一个事件event)

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

两个方法区别：

1. event_base_loopexit(base, NULL) 如果当前正在为多个活跃事件调用回调函数，那么不会立即退出，而是等到所有的活跃事件的回调函数都执行完成后才退出事件循环

2. event_base_loopbreak(base) 如果当前正在为多个活跃事件调用回调函数，那么当前正在调用的回调函数会被执行，然后马上退出事件循环，而并不处理其他的活跃事件了

## 2.5 event\_base的例子

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>    
#include <sys/socket.h>    
#include <netinet/in.h>    
#include <arpa/inet.h>   
#include <string.h>
#include <fcntl.h> 
 
#include <event2/event.h>
#include <event2/bufferevent.h>
 
int main() {
	puts("init a event_base!");
	struct event_base *base; //定义一个event_base
	base = event_base_new(); //初始化一个event_base
	const char *x =  event_base_get_method(base); //查看用了哪个IO多路复用模型，linux一下用epoll
	printf("METHOD:%s\n", x);
	int y = event_base_dispatch(base); //事件循环。因为我们这边没有注册事件，所以会直接退出
	event_base_free(base);  //销毁libevent
	return 1;
}
```

# 3 event事件

event_base是事件的集合，负责事件的循环，以及集合的销毁。而event就是event_base中的基本单元：事件。

我们举一个简单的例子来理解事件。例如我们的socket来进行网络开发的时候，都会使用accept这个方法来阻塞监听是否有客户端socket连接上来，如果客户端连接上来，则会创建一个线程用于服务端与客户端进行数据的交互操作，而服务端会继续阻塞等待下一个客户端socket连接上来。客户端连接到服务端实际就是一种事件。

## 3.1 创建一个事件event

```c
struct event *event_new(struct event_base *base, evutil_socket_t fd,short what, event_callback_fn cb,void *arg);
```

参数：

1. base：即event_base

2. fd：文件描述符。

3. what：event关心的各种条件。

4. cb：回调函数。

5. arg：用户自定义的数据，可以传递到回调函数中去。

libevent是基于事件的，也就是说只有在事件到来的这种条件下才会触发当前的事件。例如：

1. fd文件描述符已准备好可写或者可读

2. fd马上就准备好可写和可读。

3. 超时的情况 timeout

4. 信号中断

5. 用户触发的事件

what参数 event各种条件：

```c


# 参考
https://blog.csdn.net/initphp/article/details/41946061

