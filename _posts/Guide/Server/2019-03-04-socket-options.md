---
layout: post
title: 套接字选项
date: March 14, 2019 2:03 AM
excerpt: 使用套接字进行网络编程, 不能忽视的重要部分就是套接字选项的内容
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

使用套接字进行网络编程, 不能忽略的就是:**套接字选项的设置**.
设置合适的套接字选项, 可以使我们的程序有合理的行为以及性能上的提升

## 套接字选项API

首先介绍,进行套接字处理的API, 只有`setter`与`getter`
```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
```

解释一下其中的参数:
- `sockfd`: 进行设置的sockfd
- `level`: 协议界别的设置, 如`SOL_SOCKET`, `IPPROTO_TCP`等
- `optname`: 选项名, 如`SO_REUSEADDR`, `SO_RCVTIMEO`等
- `optval`: 选项值, 有的是T/F, 有的需要赋值, 所以是`void *`类型, 且为值-结果参数(对`get`)
- `optlen`: 选项大小, 对应于`optval`, 因为`void *`无法获取类型长度

这两个API分别用于套接字选项的设置和获取, 接下来介绍常用的套接字选项

## 通用套接字选项

这一部分是通用的套接字选项, 所有套接字都适用.

但是我们要先清楚, **从`accept(2)`中获取到的`connfd`是会继承套接字状态的.**
这些套接字选项是会被继承的: `SO_DEBUG`, `SO_DONTROUTE`, `SO_KEEPALIVE`, `SO_LINGER`, `SO_OOBINLINE`, `SO_RCVBUF`, `SO_RCVLOWAT`, `SO_SNDBUF`, `SO_SNDLOWAT`, `TCP_MAXSEG`, `TCP_NODELAY`;

连接套接字会继承监听套接字的上述选项.

### SO_ERROR

使用此选项, 表示获取套接字上的错误, 如果有错误发生, 套接字变为可读可写, 如果有信号驱动, 会触发`SIGIO`, 我们最常见的用法在于, 非阻塞`connect(2)`中检测状态
**此套接字选项, 是可以获取, 但是不能设置的**

### SO_KEEPALIVE

使用此选项, 启用TCP的保活机制, 两小时内, 如果没有数据交换, 设置此选项端会自动发送一个`保活探测分节`, 我们进行服务端开发时, 一般给监听套接字, 都设置此选项.

```cpp
int val = 1;
int ret = ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(int));
```
可能会收到三种响应:
1. 期望的`ACK`
2. 对端崩溃或关闭, 所引发的`[RST]`
3. 无任何响应, 则过一段时间重发, 直至关闭套接字

*注意区分: *: `ECONNREFUSED`, `ETIMEOUT`, `EHOSTUNREACH`/`ENETUNREACH`
**保活选项: 主要是用来处理, 占用资源的半开连接, 处理对端崩溃的情况**

下面分析一下TCP状态的检测方法:
- 正在发送数据时
	- 对端进程崩溃: 发送`[FIN, ACK]`, 会使得本端套接字可读, 触发`EPOLLRDHUP`, 或者`read = 0`, 如果继续写, 第一次会触发,`[RST]`, 下一次,内核会发送`SIGPIPE`
	- 对端主机崩溃: 本端将会超时, 最后`ETIMEOUT`
	- 对端主机不可达: 超时, 最后`EHOSTUNREACH`
- 正接收数据时:
	- 对端进程崩溃: 收到`[FIN, ACK]`, 作为EOF读入处理
	- 对端主机崩溃: 停止接收
	- 对端主机不可达: 停止接收
- 空闲连接, 有`SO_KEEPALIVE`
	- 对端进程崩溃: `[FIN, ACK]`正常结束
	- 对端主机崩溃: 毫无动静2小小时, 保活机制启动, 最后`ETIMEOUT`
	- 对端主机不可达: 同上,,,最后`EHOSTREACH`
- 空闲连接, 无`SO_KEEPALIVE`
	- 对端进程崩溃: 收到`[FIN, ACK]`正常结束
	- 对端主机崩溃: (无)
	- 对端主机不可达: (无)

最后的两个`(无)`就是我们面临的,半连接僵死的情况, 一般不完全依靠底层, 就可以实现用户层面的心跳协议即可. 不过没有保活机制, 基本上是死绝了

### SO_LINGER

本选项制定了`close(2)`对于面向连接的协议如何处理(`TCP`或`SCTP`, 无`UDP`)

```cpp
struct linger {
  int l_onoff;
  int l_linger;
};
```

对于这个结构体有这样的配置:
1. `l_onoff == 0`, 表示关闭`SO_LINGER`, `close(2)为默认行为`
2. `l_onoff != 0 && l_linger == 0`, 表示`close(2)`的时候, 直接关闭连接并且丢弃数据. 并且发送一个`RST`给对端, **会取消**`TIME_WAIT`状态, 可能会有*化身*的问题出现
3. `l_onoff != 0 && l_linger != 0`, 表示`close(2)`的时候, 将会等待一段时间, 发送完数据/延滞时间到, 之后丢弃所有数据, 此时检测`close(2)`返回值很重要, 若为`EWOULDBLOCK`表示:**是延滞时间到, 数据被丢弃**

虽然`SO_LINGER`选项看着很美好, 但是不尽如人意, 你无法保证是否对面确认数据
`l_linger`时间太短没用, 太长了降低效率.

所以, 建议使用`shutdown(2)`
```cpp
#include <sys/socket.h>
int shutdown(int sockfd, int how);
```

其中`how`有三种行为: `SHUT_RD`, `SHUT_WR`, `SHUT_RDWR`
- `SHUT_RD`: 表示读端关闭, 不能再从连接上读取数据, 可以继续发送, 但所有读到的数据丢弃
- `SHUT_WR`: 表示写端关系, 不能往连接上写数据, 可以继续读取
- `SHUT_RDWR`: 同默认的`close(2)`行为

### SO_RCVBUF / SO_SNDBUF

众所周知, TCP底层为收发各自维护了一个缓冲区. 
**而且, TCP的接收缓冲区永远不可能溢出, 因为滑动窗口机制, 如果对端无视滑动窗口, 数据会在本端被丢弃, 此即为TCP流量控制. 同时UDP是没有流量控制的, 很容易导致数据被淹没丢弃**

那么, **这两个套接字选项就是用来设置, 底层缓冲区大小的**

有两点十分重要的内容:
1. `SO_RCVBUF`和`SO_SNDBUF`的设置, 客户端要在`connect(2)`之前, 服务端要在`listen(2)`
 这是因为: 滑动窗口的确认是在`[SYN, ACK] -> [SYN] -> [ACK]`的三路握手过程中完成的
 所以, 设置选项一定在三路握手前, 不然是无效的, 连接套接字可以继承监听套接字的设置

2. 缓冲区大小一般是`MSS`的四倍, 为了激活`TCP快速恢复算法`, 因为连续三个重复确认, 判断分节丢失

### SO_RCVLOWAT / SO_SNDLOWAT

此两个套接字选项是设置低水位标志使用的.

低水位标志的意义在于: 最少字节触发量, 如X字节触发写事件, Y字节触发读事件

### SO_RCVTIMEO / SO_SNDTIMEO

此两个套接字是用来设置IO的超时时间, 精度与`select(2)`的`TIMEOUT`一致, 基本达到微妙定时

使用`struct timespce`设置

### SO_REUSEADDR / SO_REUSEPORT

总结一下: 可以完全使用`REUSEPORT`取代`REUSEADDR`

允许进行重复捆绑, 甚至支持完全重复的捆绑, **五元组完全相同**

这两个套接字有什么用处呢?
1. 调试方便, 可以直接关闭服务器重新启动, 无需顾虑`EADDRINUSE`
2. `REUSEPORT`的负载均衡模式

`REUSEPORT`中支持热备份模式(之前), 负载均衡模式(现)
热备份模式: 是防止意外崩溃, 在同一端口绑定不同的实例
负载均衡模式: 内核层面进行负载均衡, 减轻负担

## TCP套接字选项

其中最核心的就是`TCP_NODELAY`选项了

开启此选项将禁用TCP的`Negle`算法, 默认开启.
此算法目的在于减少局域网上的分组数量.它会尽量发送最大大小的分组,**避免同一时刻有多个待确认分组**

`Nagle算法`一般和`ACK延滞算法`合用, `ACK延滞算法`是延迟`ACK`的发送, 尽量希望其被分组捎带
就可以减少一个网络中的交换分组.

那么, 我们一般开发的网络编程, 对于此选项都是开启的, **需要禁止Negle算法**
因为我们的需求是**高并发**, **快速响应**.

一般使用`writev`或者手动输入到同一块buffer中, 然后`write`, 最不建议禁用Nagle算法.
因为, *大量小分组有损于网络*

因为`SCTP编程`接触较少. 同时IP套接字选项中也没有特别优先级高的

所以, 我们就暂且介绍上面这些重要, 且常见的套接字选项.
