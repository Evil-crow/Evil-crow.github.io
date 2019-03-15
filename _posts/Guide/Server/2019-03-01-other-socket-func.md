---
layout: post
title: 网络编程中的IO函数
date: March 13, 2019 9:14 PM
excerpt: 只使用socket API仅仅能完成网络的连接. 还需要掌握常用的IO函数, 和其他工具函数
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

## 常用IO函数

对于常用的IO函数, 我们可以这样分类:

```cpp
#include <unistd.h>                                                   (1)
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);

#include <sys/uio.h>                                                  (2)
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

#include <sys/types.h>                                                (3)
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                struct sockaddr *src_addr, socklen_t *addrlen);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
```

其中的操作函数可以分为三类:
1. 基本IO函数
2. 特殊IO函数, 分散读/集中写, 还有其原子操作版本(`preadv`)
3. 高级IO函数, 可以进行选项的设置

比较结果:

|函数|任何描述符|仅适用于套接字|单个缓冲区|分散/集中|可选标志|可选对端地址|可选控制信息|
|---|---|---|---|---|--|---|---|--|
|read, write|√| |√| | | | | 
|readv, writev|√| | |√| | | | 
|recv, send| |√|√| |√|| |
|recvfrom, sendto| |√ |√||√|√||
|recvmsg, sendmsg| |√| |√|√|√|√|

我们在IO函数中还能够可设置的标志位有: `MSG_DONTROUTE`, `MSG_DONTWAIT`, `MSG_WAITALL`, `MSG_PEEK`, 
分别为: 绕过路由表检查, 本次操作非阻塞, 等待所有消息, 窥看带外数据

谈到这里, 我们就有两个比较值的讨论的问题:

### 如何设置套接字超时

在`<UNP V1> Chapter XIV`中讨论了这个问题:

1. 使用信号进行超时, 调用`alarm(2)`产生`SIGALRM`信号
2. 使用IO复用接口进行超时控制, `timeout`参数
3. 使用`SO_RCVTIMEO`/`SO_ENDTIMEO`控制套接字超时选项

其中不推荐一, 尤其是在多线程环境下, 使用信号无疑是自寻烦恼.
使用`timeout`的精度不够十分准确, 建议阻塞套接字可以使用套接字选项进行控制, 
然而实质上都是使用`非阻塞套接字更多一点`

### 如何查看排队的数据量

需求就是: 在不确实读写套接字的条件下, 如何确定套接字中的数据排队量
1. 如果目的在于避免IO阻塞, 那么我们直接可以使用非阻塞IO即可
2. 如果要查看数据, 而且此次不读出数据, 可以使用`NonblockIO + MSG_PEEK`或者`MSG_DONTWAIT + MSG_PEEK`进行操作, 不过真正读的时候, 数据量不一定相等, 因为可能会再次收到一些数据
3. 某些实现可以使用`ioctl`的`FIONREAD`命令返回接收队列中的字节数

## 其他网络编程函数

下面介绍几个经常使用的网络编程API:
```cpp
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);

```

上面成为字节序函数, 用于完成主机和网络字节序之间的相互转换, 并分为long和short

**一般用于:** 将`in_port_t`类型的端口号, 转为网络字节序时使用

```cpp
#include <strings.h>
void bcopy(const void *src, void *dest, size_t n);
void bzero(void *s, size_t n);
int bcmp(const void *s1, const void *s2, size_t n);

#include <string.h>
void *memcpy(void *dest, const void *src, size_t n);
void *memset(void *s, int c, size_t n);
void *memmove(void *dest, const void *src, size_t n);
```

上面的几个为字节处理函数, 用于字节置位, 字节拷贝, 比较操作. 分别是BSD风格和ANSI C风格
*建议使用ANSI C风格的API*

```cpp
#include <arpa/inet.h>
int inet_pton(int af, const char *src, void *dst);
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```
上面两个是最新的地址转换函数, 用于将IP地址从ANSI字符串转为网络字节序.
之前的3个旧的API可以废弃了, 完全使用新的API即可.

## 获取端地址

有时候为了进行志记功能,我们需要获得本端,或者对端的address信息,常用的是这两个函数:
```cpp
#include <sys/socket.h>
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

使用方法是相同的, 但是一般情况下,`getsockname(2)`足够了, 因为我们一般仅仅是获取本端地址需要使用函数, 对端地址一般在`accept4(2)`的参数中获取, 节省一次系统调用.

## 网络地址信息API

这部分API使用来进行`name`与`service`之间的转换的.
对于IPv4,可以单独使用`gethostbyname`, `gethostbyaddr`, `getservbyname`, `getservbyaddr`

支持IPv4的函数有这么两个`gethostbyname(2)`, `gethostbyaddr(2)`.但是如今我们有了更好地选择`getaddrinfo(2)`和`getnameinfo(2)`可以完全支持IPv4和IPv6

```cpp
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *node, const char *service, 
              const struct addrinfo *hints,struct addrinfo **res);

void freeaddrinfo(struct addrinfo *res);
int getnameinfo(const struct sockaddr *addr, socklen_t addrlen,
                       char *host, socklen_t hostlen,
                       char *serv, socklen_t servlen, int flags);
```
要注意的是: **此函数中使用动态内存分配, 所以需要使用完后进行`freeaddrinfo(2)`的释放**

平时比较常见的其他网络编程接口基本就是这么些了.
