---
layout: post
title: 服务器程序规范
date: March 14, 2019 3:00 PM
excerpt: 除了网络通信以外, 服务端程序还有许多需要遵守的规范
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

服务端的开发有广而杂的细节, 我们着重讨论以下几个方面:

## 日志

日志这一部分内容, 是重中之重.那么我们一般分为这两类日志:
1. 本地日志
2. 远端日志

远端日志更多的做的是,同步备份的工作, 并不建议写远端, 有可能因为意外情况导致崩溃
一般使用系统级别的日志是`syslog(2)`.但是效率一般, 使用晦涩
实际上建议可以使用著名的日志库即可,Google的开源日志库, `Boost`中的日志库, 或者其他等

这里介绍本地写日志的做法:
1. 直接写, 可能会阻塞IO
2. 无缓冲地写, 使用背景线程收集日志

[阻塞日志](https://github.com/Evil-crow/platinum/blob/master/src/utility/logger.cc)
[无缓冲日志](https://github.com/Evil-crow/LevelNet/tree/master/src/utility/logger)

## 用户信息

- `UID`: 用户ID
- `EUID`: 有效用户ID
- `GID`: 用户组ID
- `EGID`: 有效用户组ID

其中区分真实用户ID和有效用户ID. **有效用户ID是为了资源访问**
我们可以以`root`身份启动, 之后切换为普通用户, 但是使用有效用户ID用于资源访问

比如,`su`命令的时候, UID不同时, `su`具有EUID, 使得他有权限进行资源访问

我们可以使用下面这些API进行用户切换
```cpp
#include <unistd.h>
#include <sys/types.h>
uid_t getuid(void);
uid_t geteuid(void);
gid_t getgid(void);
gid_t getegid(void);
int setuid(uid_t uid);
int setgid(gid_t gid);
int seteuid(uid_t euid);
int setegid(gid_t egid);
```

## 进程间关系

分析进程, 进程组, 会话这几个概念之间的关系:
这几个关系是这样: 
进程是OS调度的最小单位, 
多个进程属于同一进程组, 有进程组ID, `PGID`
多个有关联的进程组, 组成会话.

使用`PS`命令可以查看进程之间的关系

## 资源限制

资源限制是服务端开发很重要的一个内容:
- 可以使用命令`ulimit`进行设置
- 可以使用函数`getrlimit/setrlimit`在程序中进行设置

我们常见的资源限制有这么几种:
- 核心转储文件大小
- 数据段大小
- 调度优先级
- 文件大小
- 可等待信号数目
- 最大虚拟内存
- 可打开文件数
- 管道大小
- POSIX消息队列长度
- 实时调度优先级
- 栈段大小
- 用户可创建最多进程数目
- 虚拟内存
- 文件锁

## 改变目录

有时候需要进行目录的切换, 以此来进行访权限制等.有下面3个API可用:
```cpp
#include <unistd.h>
char *getcwd(char *buf, size_t size);
int chdir(cosnt char *path);
int chroot(const char *path);
```
顾名思义, 不需要进行更多的解释了.

## 后台化

一般情况下我们的服务端程序都是在后台运行的.
常见的后台化方式有这么几种:
1. 运行程序时, 使用`&`, 如: `./platinum &`
2. 自行编写函数进行后台化
3. 使用Linux提供的`int daemon(int nochdir, int noclose)`完成后台化
4. 使用`inetd/xinetd`服务器启动, (现在已经是`systemd`了)

简要分析一下后台化的步骤:
1. `fork`, 杀死父进程, 保证自己有进程组ID, 但与自ID不同
2. `setsid`, 脱离控制终端
3. 忽略`SIGHUP`(会话头进程终止, 所有会话都会有此信号), 并再次`fork`
4. 为错误处理函数设置标识
5. 改变工作目录
6. 关闭所有打开的描述符
7. 将`stdin`, `stdout`及`stderr`重定向到`/dev/null`
8. 使用`syslogd`处理错误

但是, 我们直接使用此函数即可

```cpp
#include <unistd.h>
int daemon(int nochdir, int noclose);
```

> The daemon() function is for programs wishing to detach themselves from the controlling terminal and run in the background as system daemons.


