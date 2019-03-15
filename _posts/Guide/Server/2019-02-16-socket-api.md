---
layout: post
title: 基本的scoket APIs
date: March 13, 2019 5:32 PM
excerpt: 服务端编程的基础就是使用socket API进行网络通信.
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

我们进行服务端编程的基础就是使用socket系列API. 那么, 其中有哪些坑或者要注意的点?

## socket(2)

使用套接字的第一步就是, 创建套接字.创建套接字的API是`socket(2)`
```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```
其中`<sys/types.h>`仅仅是BSD系列套接字需要的, 遵循`POSIX`标准的Linux不需要此头文件
通过设定`domain`, `type`以及`protocol`从而创建套接字.
```cpp
domain: AF_UNIX(AF_LOCAL), AF_INET, AF_INET6等
type: SOCK_STREAM, SOCK_DGRAM, SOCK_RAW, SOCK_SEQPACKET等
protocl: 一般指定为0, 因为type一般与protocol是唯一对应的, 仅仅在少数情况下, 需要指定协议
```
在`type`上可以进行套接字选项的设定, 一般特指这两种:
```cpp
SOCK_NONBLOCK: 同O_NONBLOCK, 将套接字置为非阻塞套接字, 默认套接字是阻塞的
SOCK_CLOEXEC: 同O_CLOEXEC, 原子性的设置O_CLOEXEC, 在exec(2)的时候, 关闭设置此选项的fd
```

返回值: 成功为0, 错误发生为-1, 且设置`errno`

## bind(2)

在创建一个套接字之后, 我们需要将套接字绑定在一个地址上, 这样才能在端口上开放服务
绑定套接字的API是`bind(2)`

```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

`bind`操作可以说就是"给套接字赋予一个名字(assign a name to socket)"

因为, socket API出现的太早, 且为了兼容, 所以使用了公共的指针描述, `struct sockaddr *`,
放在现在完全可以使用这样的声明`int bind(int sockfd, const void *addr, socklen_t addrlen);`
所以, 对于不同的协议, 使用不同的地址描述, 基于`domain`
```cpp
AF_INET => struct sockaddr_in;
AF_INET6 => struct sockaddr_in6;
...
```

我们在使用`bind(2)`时, 需要设置`addr`和`port`, 这两者未指定时, 便由内核进行指定.
我们仅讨论对服务器的情况:
如果端口未指定, **则会由内核挑选一个临时端口**, 问题就是: **客户端无法得知服务器在那个端口上服务**
所以, 我们一般都**要求(必须)指定端口的.**

另一方面, 我们可以将IP地址设为通配地址, 这样有什么好处呢?
**对于具有多块的网卡, 我们的服务可以不限制地址, 在多个网卡上服务**
**服务器, 会在收到第一个该端口上的SYN中确定服务应该绑定的地址**

对于IPv4, 使用`INADDR_ANY`常值作为通配地址.

返回值: 成功为0, 错误发生为-1, 且设置`errno`

常见错误: `EADDRINUSE`, 我们可以使用套接字选项 `SO_REUSEADDR`/`SO_REUSEPORT`

使用实例:
```cpp
// man 2 bind
#define handle_error(msg) \
           do { perror(msg); exit(EXIT_FAILURE); } while (0)
struct sockaddr_un my_addr, peer_addr;
socklen_t peer_addr_size;

sfd = socket(AF_UNIX, SOCK_STREAM, 0);
if (sfd == -1)
  handle_error("socket");

memset(&my_addr, 0, sizeof(struct sockaddr_un));
                               /* Clear structure */
my_addr.sun_family = AF_UNIX;
strncpy(my_addr.sun_path, MY_SOCK_PATH,
sizeof(my_addr.sun_path) - 1);

if (bind(sfd, (struct sockaddr *) &my_addr,
  sizeof(struct sockaddr_un)) == -1)
  handle_error("bind");
```

**注意: ** 服务端一般显示`bind(2)`, 因为要开放端口给客户端连接. 而客户端一般是发起连接的, 并不进行显示的`bind(2)`, 由OS在`connect(2)`之前, 进行隐式`bind(2)`. C/S模型是对等的, 所以客户端也需要`bind(2)`自然而然就能理解了

**注意: ** 服务端使用`bind(2)`之后, TCP状态: `CLOSED -> SYN_RECV`
客户端使用`bind(2)`之后, TCP状态: `CLOSED -> SYN_SENT`

## listen(2)

在给一个套接字具名之后, 就可以使其处于监听的状态了. 使用的API是 `listen(2)`

```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```
`listen(2)`套接字API比较简单, 是套接字处于监听状态(监听是否有套接字对其发起连接)
那么, 这个API中要注意的是什么呢? 是的, 正是其中的参数`backlog`

如果有新连接到来的时候, 此队列已经full, 则客户端会收到`ECONNREFUSED`,会被拒绝连接.
在OS底层实现中, 为每个监听套接字维护了两个队列:
1. 未连接队列: 其中的套接字都是收到`[SYN, ACK]`,发送`[SYN]`后, 等待对端`[ACK]`的套接字
2. 已完成队列: 其中的套接字都是收到对端`[ACK]`的三路握手完成队列, 处于`ESTABLISHED`状态

从此处就可以得知: **TCP三路握手的完成与`accept(2)`无关, `accept(2)`仅仅是从完成队列中获取**
`backlog`的参数含义比较抽象: BSD实现中将其定义为此两个队列长度之和.
而且还给定了一个模糊因子: *backlog * 1.5即为等待队列的长度*

Emmm, 我们现在使用的都是POSIX标准的API, 可以查看`man 2 listen`可知:
`backlog`就是指: **待连接队列长度, 即等待三路握手完成的队伍, 处于`SYN_RCVD`状态**
>  The backlog argument defines the maximum length to which the  queue  of  pending connections for sockfd may grow.

那么, `backlog`设置多大合适呢? 不同的系统有不同的算法.
我们一般可以直接使用`SO_MAXCONN`即`int listen_fd = ::listen(sockfd, SO_MAXCONN)`

在Linux内核2.2之后，分离为两个backlog来分别限制半连接(`SYN_RCVD`状态)队列大小和全连接(`ESTABLISHED`状态)队列大小。

即, 不是用一个`backlog`来进行计算.

`SYN队列`长度由`/proc/sys/net/ipv4/tcp_max_syn_backlog`指定，默认为2048。
`Accept队列`长度由`/proc/sys/net/core/somaxconn`和使用listen函数时传入的`backlog`，
二者取最小值。默认为128。
原来是写死的代码`SO_MAXCONN`, 现在可以使用`/proc/sys/net/core/somaxconn`进行设定, 或者在`/etc/sysctl.conf`中设置`net.core.somaxconn = xxx`;

**注意: **`listen`使得套接字从`CLOSED -> LISTEN`状态转变

## accept(2)

从上面一连串的系统调用下来, 就到了`accept(2)`. 如我们所知, `accept(2)`并非三路握手的必需
**仅仅是从完成连接队列, 即Accept队列中返回已经完成的连接**

当队列为空时, 阻塞模式会陷入睡眠, 非阻塞模式会返回`EAGAIN`
```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sys/socket.h>
int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);
```

在2.6之后的版本, 现在提供了新的`accept4(2)`接口, 其便利之处在于: `flags`可直接设置
此处`flags`的可选项为`SOCK_NONBLOCK` + `SOCK_CLOEXEC`

**节省一次系统调用, 是提高效率的便利方案**

这里面,`addr`是典型的值-结果参数. 它表示从内核中获取我们所需要的信息.所以是指针, 当我们不关注对端的地址信息时, 可以将其设置为`NULL(nullptr)`

**注意: **`accept(2)`并不能使套接字的TCP状态发生转变, 因为`SYN_RECV -> ESTABLISHED`是发生在待连接队列中的, 完成的时候, `accept(2)`不一定被调用

## connect(2)

上面的API都是进行服务端构建所使用的. 这里介绍一个客户端所必须使用的API, `connect(2)`
```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

我们一般不要求客户端绑定端口, `connect(2)`中填入的是对端(服务器)的地址.
**因为内核会确定源IP地址, 并选择一个临时端口号作为源端口进行`bind(2)`**

那么, `connect(2)`中一般常见的是这几个错误:
1. `ETIMEOUT`, 因为收不到对端的`[SYN]`, (已经发送`[SYN, ACK]`), 逐次超时重传仍无效, 关闭连接
2. `ECONNREFUSED`, 是客户端收到返回的`[RST]`分节造成, 属于**硬错误**
3. `EHOSTUNREACH`/`ENETUNREACH`, 这是属于`ICMP`错误, 是路由不可达的原因, 属于**软错误**
 可能是本机路由转发表的对端路径根本不可达, 或是`connect(2)`直接返回

**注意: **使用`connect(2)`使得TCP套接字从`CLOSED -> SYN_SENT` 状态, 成功后是`ESTABLISHED`状态

常用的socket API基本就是这些. 之外还存在着`getpeername(2)`, `getsockname(2)`等辅助函数, 用来获取到对端, 本端地址地址的, 至于是否是线程安全函数, 看具体实现

## close(2)

无论对于客户端, 还是服务器, 连接的关闭都是必不可少的.因为我们使用套接字都是用文件描述符, 所以,
可以使用统一的系统编程接口去关闭连接`close(2)`
```cpp
#include <unistd.h>
int close(int fd);
```
所有用法同任何文件描述符相同, 我们要注意的是: `close(2)`会将套接字标为关闭
而众所周知, TCP连接是全双工的, 所以就有半连接的概念, 具体有`SO_LANGER`和`shutdowm(2)`可处理

使用以上API, 我们已经能够构建出简单地`Echo`, `PingPong`, `daytime`等简单地网络程序了.
