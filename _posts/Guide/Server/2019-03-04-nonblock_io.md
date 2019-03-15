---
layout: post
title: 非阻塞IO
date: March 14, 2019 10:09 PM
excerpt: 之前我们在IO复用中提到过, 非阻塞IO单独使用就是在作死, 配合上IO多路复用机制才会体现出它的作用来, 本篇我们就来分析非阻塞IO的各种操作
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

在讲到非阻塞IO的时候, 我们经常涉及的接口就是: (仅涉及网络编程范围内)
可能阻塞就分为四类:
1. 输入操作: `read`, `recv`, `recvfrom`, `readv`, `recvmsg`
2. 输出操作: `write`, `send`, `sendto`, `sendv`, `sendmsg`
3. 发起对外连接: `connect`
4. 接受外来连接: `accept`

输入输出操作, 如果能非阻塞的操作下去, 就会直接操作. 否则, 会立即返回`EAGAIN/EWOULDBLOCK`避免阻塞.

我们着重来分析, 对于`connect`和`accept`的非阻塞行为:

## accept

大多数的情况下: 在我们使用多路复用配合 + 非阻塞IO的时候.
**只要文件描述符返回, 它一定是不会阻塞的, 一定是可以直接接受连接的**

但是, 历史上存在这样的问题: (现在好像还是有, 我测试了, 结果有问题)

客户端向服务端发起连接, IO复用记录该套接字就绪, 在返回之前, 客户端宕机,(或者其他情况, 都行)
这时候, 直接收到`[FIN, ACK]`连接断开了.
源自BSD实现的`scoket API`会将这个错误不进行报告, 直接在内核层面处理掉这个套接字fd
然后, 基于POSIX的实现, 现在会导致**监听套接字, 阻塞在**`accept(2)`上

虽然这个问题也不是很严重, **一旦有一个新的连接, 阻塞状态立即解除**
一般返回的错误标志是`ECONNABORTED (SysV)`,或者`EPROTO (POSIX)`

解决办法有很简单:
1. 总是将监听套接字设置为非阻塞的.
2. 同时忽略`EWOULDBLOCK`, `ECONNABORTED`, `EPROTO`, `EINTR`

设置为非阻塞监听套接字还有一个间接原因是: 
**连接套接字会直接继承监听套接字的属性**, 减少了一次`fcntl`系统调用

当然,现在使用`accept4(2)`也可以直接避开这个问题.

## connect

另一个比较重要的就是`connect`的非阻塞调用了.
我们回顾一下基础的网络知识: 
TCP三次握手的流程是这样的: `[SYN,ACK] -> [SYN] -> [ACK]`
三路分组交换, 客户端发送`[SYN, ACK]`之后, 它需要等待一个RTT, 而在这个RTT之间我们可以作很多事.
我们也是因此才会有使用`Nonblock connect`的理由

那么, `非阻塞connect`的注意点在哪里呢?
1. `connect`返回`EAGIN`, `EWOULDBLOCK`是真正的错误, 必须关闭套接字
2. `connect`返回`EINPROGRESS`才是真正的非阻塞情况

我们进行`非阻塞connect`的判断需要结合IO复用,当`connect`无论是成功还是失败
都会触发读写事件, 此时我们查看套接字上发生的错误就可以判断, 是否`connect`成功

基本流程是这样的:
```cpp
int ready = epoll_wait(ep_fd, &events, max_events, timeout);
for (...) {
  if (events[i].events & EPOLLIN &&
      events[i].events & EPOLLOUT) {
      if (getsockopt(events[i].data.fd, SOL_SOCKET, SO_ERROR, &error, sizeof(int)))
        if (error == 0)
          // connect successfully
	  }
}
```

### 被中断的connect

对于其他的系统调用, 如果被中断, 一般返回`EINTR`, 此时我们会尝试重启系统调用.
但是, 对于`connect`, **我们不能重启, 应该立即关闭套接字**. 如果强行使用, 会导致`EADDRINUSE`
