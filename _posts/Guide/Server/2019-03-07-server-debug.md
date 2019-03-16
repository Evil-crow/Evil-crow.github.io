---
layout: post
title: 网络编程性能调优与测试
date: March 16, 2019 8:22 PM
excerpt: 编写一个高性能的服务器固然重要, 对服务器的维护, 测试, 调试同样十分关键
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

前面我们已经详细分析过网络编程的方方面面, 现在我们需要从系统方面进行调优.

Linux平台有一个很重要的特性就是内核微调, 通过修改 `/proc`目录下的文件.
`/proc/sys` 目录存放所有可读写的文件的目录，可以被用于改变内核行为

比如说: 
`tcp_max_syn_backlog`可以用来修改SYN队列的最大长度
`/proc/sys/fs/epoll/max_user_watches`可以用来修改`epoll`最大观察文件描述符的数目

下面我们就来详细查看一下修改内核参数的事

## 修改内核参数

### file-max

最大文件描述符是非常非常重要的资源, **我们每一条连接就要占用一个文件描述符**
那么, 我们修改文件描述符可以从哪方面入手呢?

两方面: 全局限制和进程限制

1. 全局限制
 通过修改内核参数`/proc/sys/fs/file-max`,不过这个也是临时的. 同时将`/proc/sys/fs/inode-max`设为原来的3~4倍, 不然有可能会造成`inode`节点不够用

2. 进程限制
 我们上面修改的也只是总的数目, 每个进程的限制由`ulimit -n`控制, 可以修改为指定数目, 或者
 `max-file-number`, 即`ulimit -SHn max-file-number` (软硬限制一起修改)
 
为了永久进行修改, 我们需要在`/etc/security/limits.conf`文件中写入:
```bash
 # 修改进程限制
 * hard nofile max-file-number
 * soft mofile max-file-number
```

为了永久修改全局限制, 需要在`/etc/sysctl.conf`文件中这样修改:
`fs.file-max=max-file-number`, 之后执行`sysctl -p`生效

### max_user_watches

这个参数指的是, 一个用户所能向`epoll`内核时间表中所注册的事件总数, 指的是所有`epoll`实例所能监听的文件描述符总数, 而不是一个用户的, 这个参数限制了epoll使用的内核内存总量
此文件在`/proc/sys/fs/max_user_watches`

### somaxconn

这个参数主要描述的是: 连接完成队列中的最大连接数, `accept`就从此队列中取走连接
文件在`/proc/sys/net/somaxconn`

### tcp_max_syn_backlog

这个参数主要用来描述的是: SYN队列的长度, 也就是说等待连接`[ACK]`的最大连接数
文件在`/proc/sys/net/tcp_max_syn_backlog`

### tcp_wmem

这个参数主要用来描述的是: 一个socket的TCP写缓冲区的最小值, 默认值和最大值
文件在`/proc/sys/net/ipv4/tcp_wmem`

### tcp_rmem

这个参数的含义是: 一个socket的TCP读缓冲区的最小值, 默认值和最大值
文件在`/proc/sys/net/ipv4/tcp_rmem`

### tcp_syncookies

通过打开TCP同步标签, 同步标签启动`cookie`来防止监听一个socket因不停的重复接受来自同一个地址的连接请求(即`SYN`), 而导致listen监听队列溢出

上面这些修改都是临时的修改, 如果要永久修改
1. 在`/etc/sysctl.conf`中加入设置
2. 之后执行`sysctl -p`使其生效

## GDB调试

多进程:
- `attach pid`: 即可切换到指定进程
- `set follow-fork-mode [parent|child]`: 指定`fork`之后是调试子进程还是父进程

多线程:
- `info threads`: 显示所有线程, 可根据GDB给的ID, 进行切换
- `thread ID`: 切换到指定ID线程
- `set scheduler-locking [off|on|setp]`: 设置其他线程是否锁定


