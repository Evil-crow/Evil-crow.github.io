---
layout: post
title: IO多路复用
date: March 14, 2019 4:33 PM
excerpt: 进行高性能服务端的利器---IO多路复用
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

进行高性能的服务端编程, 所必不可少的就是---IO多路复用技术.
多路复用到底是在干什么? 我们简单的来说: 就是OS同时监控多个连接, 当有事件发生的时候, 进行通知
此即为IO多路复用.需要结合IO模型进行理解

我接下来们介绍几种常见的IO复用机制

首先需要明确的是IO复用进行事件通知的两种机制: **水平触发**和**边缘触发**

- 水平触发: 若文件描述符上可以非阻塞的进行IO操作, 则认为它已经就绪
- 边缘触发: 当文件描述符上的状态发生改变的时候,自上次之后发生了新的IO活动, 认为已经就绪

我们提到IO多路复用, 还要明确, 一定要配合`Nonblock IO`进行使用.
1. 非阻塞IO一般配合ET通知机制一起使用
2. 因为多路复用是检查多个文件描述符, 若阻塞在一个文件描述符上, 从而阻止检查其他的文件描述符
3. 即使写就绪, 如果写入大块数据还是会阻塞

基于以上3点理由, 我们使用IO多路复用, 必须配合`Non-block IO`

下面是三种常见的IO复用模型:

## select

`select(2)`是BSD风格的IO复用模型.现在也是`SUSV3`标准中支持的IO多路复用接口
```cpp
#include <sys/select.h>

 /* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```

以上便是`select(2)`的接口使用API.
使用`select(2)`会一直阻塞, 直到文件描述符集合成为就绪态, 或者超时.
- `nfds`: 设置为三个集合中最大集合数目 +1
- `readfds`: 用来检测输入是否就绪的文件描述符集合
- `writefds`: 用来检测输出是否就绪的文件描述符集合
- `exceptfds`: 用来检测异常情况是否发生的文件描述符集合

所有的文件描述符集合使用掩码实现, 具体是`FD_xxx`系列宏.
- `FD_ZERO`: 将集合初始化为空
- `FD_SET`: 将fd ,添加进入集合
- `FD_CLR`: 将fd, 从集合中移除
- `FD_ISSET`: 判断fd, 是否在集合中

而每个集合有最大数目限制, `FDSIZE`, Linux上为`FDSIZE = 1024`, 是硬编码

我们使用`select(2)`的流程是:
`设置FD_SET结构并初始化` `->` `将fd添加如指定集合中` `->` `设置超时时间` `->` `select返回` 
`->` `检查集合并操作`

`select`的返回是这样子的:
1. `-1`表示发生错误, 我们根据`errno`进行错误判定
2. `0`表示在任何文件描述符就绪前已超时, 任何集合都会被清空
3. 正整数,表示就绪文件描述符总数. **切记: 会重复统计不同集合中的同一文件描述符**

据此, 在`select`正常返回后, 我们应该这样处理:
```cpp
int ready = ::select(nfds, &readfds, &writefds, &exceptfds, timeout);
if (ready == -1) {                         // error
  ::exit(EXIT_FAILURE);
} else if (ready == 0) {                   // timeout
  FD_ZERO(&readfds);
  FD_ZERO(&writefds);
  FD_ZERO(&exceptfds);
} else {                                   // fds ready
  for (int i = 0; i < nfds; ++i) {
    if (FD_ISSET(i, &readfds)) {
      // deal with readable event
	} else (FD_ISSET(i, &writefds)) {
      // deal with writeable event
	}
  }
}
```

总结一下, `select`调用具有这样的特点:
1. 使用三个专门的事件集合进行关注
2. 使用宏进行置位, 集合是掩码操作, 所以其占用空间小
3. 检测的fd有上限, 一般是: `FDSIZE = 1024`
4. 返回时, 只有就绪事件个数, 必须遍历每个集合, 使用指定宏`FD_ISSET`进行判断是否就绪

`System V`版本的IO多路复用, 比此有一定程度上的性能提升, 即`poll`

## poll

```cpp
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
  int fd;                // 文件描述符
  short events;          // 感兴趣的事件集合
  short revents;         // 返回的事件集合
};
```
- `fds`: `poll`对于文件描述符的集合
- `nfds`: 关注的文件描述符总数
- `timeout`: 超时时间

至于`poll`感兴趣的事件集合, 可以详细查看`man page`
`timeout`超时设定情况, `poll`和`select`行为一致;

`poll`的返回值:
- `-1`: 表示发生了错误, 有可能是`EINTR`, 表示被信号中断
- `0`: 表示在任意一个描述符就绪前, 就超时了
- 正整数: 表示就绪文件描述符数目, **不同于**`select`, `poll`**不会重复统计**

那么, 我们使用`poll`的实例是这样的:
```cpp
struct pollfd *pollFd;
pollFd = calloc(num, sizeof(struct pollfd));

for (int i = 0; i < num; ++i) {
  pollFd[i].fd = xx;
  pollFd[i].events = POLLIN | xxx | xxx;
}

int ready = ::poll(pollFd, num, -1);
if (ready == -1) {
  exit(EXIT_FRAILURE);
}

for (int i = 0; i < num; ++i) {
 if (pollFd[i].revents & POLLIN)
   // deal with readable event
 else if(pollFd[i].revents & POLLOUT)
   // deal with writeable event
 ....
}
```

那么, 我们总结一下`poll`的使用特点:
1. 优化数据结构, 使用起来比`select`明了
2. 不会重复统计就绪文件描述符数目
3. 仍然需要根据数目进行轮询判断
4. 没有上限限制
5. 可以只关注部分文件描述符

总结比较`select`与`poll`:
1. 内核层面, 使用了相同的内核`poll例程集合`, 基本同`poll`, `select`的实现是将`poll`事件转为`select`事件
2. `poll`没有1024的文件描述符上限限制
3. `select`用同一集合, 多次调用的时候, 需要多次`FD_CLR`, `FD_SET`. `poll`则是用不同的字段避免
4. `select`的超时精度高于`poll`

在性能上:
1. 当文件描述符较少的时候, `select`, `poll`都能获得不错的性能
2. 当文件描述符较多, 且分布的密集时, 性能都还行
3. 当分布的分散时, `poll`性能远高于`select`, 因为`poll`只需要检查感兴趣的文件描述符即可

`select`和`poll`的性能瓶颈:
1. 每次必须检查所有文件描述符, 耗费大量时间
2. `select`, `poll`每次都将感兴趣的事件集合拷贝进入内核, 随着感兴趣事件列表的扩增, 拷贝上的时间开销, 内存开销越来越大
3. 每次调用结束后, 必须检查返回的数据结构中的每个元素, 以此判断是否处于就绪态

实际上是因为`select`和`epoll`作为老式API的历史遗留问题, 如今我们可以放心的使用`Linux kernel 2.6`之后增加的`epoll系列`编程接口, 进行大量且高效的文件描述符管理


## epoll

`epoll`作为目前Linux服务端编程的中流砥柱, 它具有这样的优点:

同`select`和`poll`相比:
1. 大量文件描述符需要进行关注的时候, 基本不会损失太多性能
2. 即支持水平触发, 也支持边缘触发

同`Signal-driven IO`相比:
1. 避免了信号处理的复杂性, 多线程环境下使用无障碍
2. 灵活性高, 可以指定我们感兴趣的事件类型

`epoll`的编程接口如下:
```cpp
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
int epoll_pwait(int epfd, struct epoll_event *events,
                int maxevents, int timeout,
                const sigset_t *sigmask);
```

我们基本上分为三类:
1. `epoll_create`: 创建`epoll`实例
2. `epoll_ctl`: 修改兴趣列表
3. `epoll_wait`: 事件返回

### epoll_create

使用`epoll_create`创建一个epoll实例, 这是epoll的核心数据结构.
对于`epoll_create(int size)`,保证`size`为非负数即可
对于`epoll_create1(int flags)`,设置为`EPOLL_CLOEXEC`即可

### epoll_ctl

`epoll_ctl`是进行epoll实例感兴趣事件列表的修改的.基于`struct epoll_event`
```cpp
struct epoll_event {
  uint32_t events;
  epoll_data_t data;
}

typedef union epoll_data {
  void        *ptr;
  int          fd;
  uint32_t     u32;
  uint64_t     u64;
} epoll_data_t;
```
我们一般怎么用呢?
1. 设置感兴趣的事件列表, 使用`|`连接
2. 对于`data`字段, 是共用体, 我们一般设置`fd`, 或者`ptr`进行数据保存

`epoll_ctl`支持三种操作: `EPOLL_CTL_ADD`, `EPOLL_CTL_MOD`, `EPOLL_CTL_DEL`分别对应着, 将`event`添加, 修改, 删除于epoll实例中

`epoll`的事件列表, 和`poll`是类似的.
我们着重解释下面几个事件:
- `EPOLLIN`: 关注读事件
- `EPOLLOUT`: 关注写事件
- `EPOLLRDHUP`: 在ET模式下, 直接判断对端断开连接, 可以不使用`read -> 0`判断
- `EPOLLET`: 启用ET触发模式
- `EPOLLONESHOT`: 关联的文件描述符仅仅触发一次, 之后除非`EPOLL_CTL_MOD`重新激活epoll实例检查

注意一个关键点: 
`/proc/sys/fs/epoll/max_user_watches`定义了用户可注册到epoll实例的总数, 本机`1638195`

### epoll_wait

`epoll_wait`是我们最最核心的IO复用接口.
- `epfd`: 是epoll实例的文件描述符
- `events`: 是返回的事件实例数组, 这是一个值-结果参数
- `maxevents`: 上一个参数数组的大小
- `timeout`: 超时时间

对于, epoll, 我们有这样的使用实例:

```cpp
int epfd = epoll_create(3);

struct epoll_event ep_event{};
ep_event.events = EPOLLIN | EPOLLET | EPOLLRDHUP | EPOLLET;
ep_event.data.fd = fd_;

std::vector<epoll_event> ep_vec(100);
int ret = epoll_wait(ep_fd, ep_vec.data(); 100, timeout);
for (auto &var : ep_vec) {
  if (var.events & EPOLLIN)
  // deal with readable event
  if (var.events & EPOLLRDHUP)
  // deal with RDHUP
  ....
}
```

### epoll同select, poll区别

三种IO复用机制各有特点, 我们来分析`epoll`的独到之处:
1. `select`和`poll`需要内核进行调用中的所有文件描述符的检查,与之相反, `epoll`会在打开的文件描述符上下文相关联的列表中记录该描述符, 之后一旦就绪, 就在就绪列表中添加一个元素, `epoll_wait`仅仅是简单的取出这些元素
2. `select`和`poll`调用结束后, 会返回传入的文件描述符集合, 我们需要遍历所有文件描述符.其中有的可能并没有事件, 而`epoll`中用`epoll_ctl`建立了一个数据结构, 会将监视的文件描述符都记录下来, 之后就在也不需要传递任何文件描述符相关的信息给内核了, 而返回中也只是包含了处于就绪态的描述符

因此可以得出: **epoll适合处理多连接, 少活跃的网络需求**

### epoll核心: ET模式

*坊间传闻: ET是`epoll`的高效模式,真的就是这样吗?, 不尽然*

首先, 边缘触发模式, 指的是: 启用此模式的文件描述符,在有新IO活动发生之前, 不会重新通知
这就造成了**我们每次处理要, 尽可能的处理完数据**, 因为这样, 我们才能避免数据丢失

比如说: 有个文件描述符, 再也不触发事件了, 那么数据第一次没有读完, 就会永久丢失, 再不会触发了.
所以, **ET模式, 配合非阻塞套接字使用更佳哟**

据此, 我们边缘触发通知的基本框架如下:
1. 设置为非阻塞文件描述符
2. 通过`epoll_ctl()`构建`epoll实例`感兴趣列表
3. 通过如下循环处理I/O事件
 - 通过`epoll_wait`取得所有处于就绪态的描述符列表
 - 针对每一个处于就绪态德文件描述符, 不断进行I/O处理直到相关的系统调用返回`EAGAIN`或者`EWOULDBLOCK`

那么, `ET`真的无敌吗? 并不是. `ET`极有可能造成文件描述符饥饿现象:
**某一个就绪事件上是不间断的输入流** -> **造成其他文件描述符饥饿**

解法就是: 在用户层面作更多的时间控制, 避免IO操作无限进行下去.
对于这样的情况, LT更加合适, 没错吧, 因为下一次`epoll`返回, 依旧会触发通知

之所以说`ET`高效, 是因为进行事件通知的次数更少, 但是当频繁多次IO操作, 也会造成时间开销.
**因为, 要有EAGAIN, 至少要多读一次.** 
chenshuo的编码是: 读一次, 如果ret < size, 则证明下一次一定会阻塞, 可以减少一次系统调用

所以说, `LT`, `ET`究竟谁更高效,更多的是要看实际测试情况, 并不能一言以蔽之.
因此, `LT`也是`epoll`的默认工作模式, 诸多网络库 也是选择`LT`模式, 如`muduo`, `libevent`, `Boost.Asio`

下次有机会了, 可以去尝试看看`epoll`的内核源码.
