---
layout: post
title: 从C10K谈起, 聊聊服务器的设计模型(上)
date: March 15, 2019 6:18 PM
excerpt: C10K是每一个网络编程学习者都要了解的问题, 我们就从C10K谈起, 来说说各种服务端的编程模型
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

C10K问题是个缩写, 其实就是`Client 10 000`问题, 也就是`单机1w并发连接`的解法
那么, 接下来我们就从各种网络编程模型谈起, 最后来说说怎么解决单机1w+并发连接的问题.

说到网络编程的模型, 无非是这两种:
1. 一个进程/线程服务一个连接
2. 一个进程/线程服务多个连接

我们现在从这两种类型开始, 分析现有的网络编程模型:

## 迭代服务器

迭代服务器是最最古老的网路编程模型, 大概在网络服务刚开始时使用
基本思路: 服务器等待连接, 有连接到来就为其服务, 服务结束后重新等待连接.

代码描述
```cpp
socket = new Socket;
socket.bind(addr);
socket.listen;

for ( ; ; ) {
  connfd = socket.accept;
  // deal with connection
}
```
一般而言, 迭代服务器中使用的都是**阻塞IO**, 因为需求足够了.

优点: 用户使用体验良好
缺点: 服务期间其他用户需要排队, 如果并发量过高超出`SYN队列`,甚至会拒绝连接

## 并发服务器

迭代服务器的应用范围及其有限, 基本上没多久没销声匿迹了.更主流的就是并发服务器了
并发模式是两种: 多进程和多线程

### 多进程并发: Fork

最单纯的进程并发就是, 每来一个客户, 为其准备一个进程.

代码描述
```cpp
socket = new Socket;
socket.bind(addr);
socket.listen;

for ( ; ; ) {
  connfd = socket.accept;
  pid = fork;
  if (pid == 0) {
    // deal with connection;
  } else {
    // continue father process;
  }
}
```
此同样是使用**阻塞IO**

优点: 相比于迭代服务器, 支持的并发量有明显提升(如果是CPU计算密集型)
缺点: 每个连接一个进程, 开销巨大, 基本上是拿资源换响应, 如果并发量相当大, 甚至会程序崩溃, 同时,现场`fork`的开销, 相当巨大, 比`CreateProcess`多一截. 下面我们就避免现场`fork`的开销

### Pre-forking: accept无锁保护

上面是最粗糙的进程并发, 基本不能直接进入生产环境, 比较成熟的方式是: `pre-forking`
提前准备一些进程, 然后在每个子进程中进行迭代式的服务

代码描述
```cpp
int children_func(i, listenfd, addr) {
  if ((pid = fork) > 0)
    return pid;
  child_main(i, listenfd, addr);
}

void child_main(i, listenfd, addr) {
  for ( ; ; ) {
    connfd = listenfd.accept(addr);
    // deal with connection
  }
}

main() {
  socket = new Socket;
  scoket.bind(addr);
  socket.listen;

  for (seq(x, y)) {
    *pid++ = children_func(i, listenfd, addr);
  }
  
  wait(...);
}

```

这种方式就像对于上面一种更为成熟, 因为它使用了`Pre-forking`技术, 减轻了现场`fork`的开销.
但是, 这种处理方式的问题在于:
1. 首先不好预估准备的子进程数目.能否动态的进行控制也是问题所在
2. 因为`accept`没有锁的保护, 所以会出现`"惊群"`这个问题(伪概念).

第一个问题有解决的成熟方案: **进程池**即可
第二个问题: 惊群

其实在网络编程中广泛存在: `accept`, `IO复用`, `线程池`等等中都会有这个问题
本质问题就是: 事件就绪导致多个进线程唤醒, CPU上下文切换开销增加, 导致性能受损

解法比较直接: 1. 锁保护 2. 使用`REUSPORT`, 具有内核层面的负载均衡模式
其中`accept`的惊群, 在内核2.6之后已经解决, 使用条件等待标志, 有连接到来, 只唤醒等待队列上的第一个进/线程, 已经有效的解决这个问题了.即使是`Nginx`中的`epoll惊群`也是使用锁解决的

有关`REUSEPORT`可以看看之前的关于套接字选项的详解

### Pre-forking: accept使用文件锁

就像上一节中提到, 需要锁保护来解决惊群问题. 我们这里展示使用文件锁的情形

文件锁是一种使用临时文件来控制的锁机制, `fcntl`进行上锁与解锁
现在文件锁真的用的不多(2019年), 在此不详细展示.

相比于上一种示例的方式: 在`accept`前后上锁, 解锁即可完成对`accept`的锁保护

```cpp
main() {
...
file_lock_init;
for (seq(x, y)) {
  pids[i] = children_func();
}
}

child_main() {
...
file_lock_lock();
conn = listenfd.accept();
file_lock_release();
....
}
```
实现上就是将阻塞从`accept`转移到`file_lock`上, 这样就不会导致惊群现象了

### Pre-forking: accept使用线程锁

同上理, 不过使用线程锁替换了文件锁, **因为文件锁涉及文件系统, 效率会差**

### Pre-forking: 传递描述符

这种模型是使用管道将描述符传递给子进程, 使用IO复用进行监测描述符的变化.
因为, 实用性差, 复杂, (现在真的没有什么意义...)
接下来我们详细看看如何使用线程

### 多线程并发: pthread

`Stevens`先生直接说: 如果主机支持线程, 我们直接用子线程取代子进程
最简单的版本, 同多进程并发的原型一样, 每个连接一个线程处理.

代码描述:
```cpp
socket = new Socket;
socket.bind(addr);
socket.listen;

for ( ; ; ) {
  connfd = socket.accept;
  auto t = std::thread([](connfd) { thread_func(connfd); });
  t.join();
}


thread_func(connfd) {
  // deal with connection affair;
}
```

这种方式简单易行. 当然和之前的问题一致: 现场创建线程的开销也不小.

### Pre-threading: 每个线程各自accept, mutex保护

很明确的, 我们可以预先创建多个子线程, 然后在每个子线程中各自`accept`
这部分很明确,我们在每个线程中`accept`,但是同时只能有一个线程`accept`连接, 所以需要互斥锁保护
代码示例:
```cpp
main() {
  socket = new Socket;
  socket.bind(addr);
  socket.listen;

  for (int i = 0l i < num; ++i) {
    pthread_create(...);
  }
}

pthread_mutex_t lock = PTHREAD_MUTEX_INIT;

thread_func() {
  for ( ; ; ) {
    pthread_mutex_lock(&lock);
	connfd = listenfd.accept;
	pthread_mutex_unlock(&lock);
    // deal with connection
  }
}
```

不加锁也可以, 便会从线程库调度转为内核调度, 反而会增大开销, 降低效率

### Pre-threading: 主线程统一accept

与之前的区别就是, 指定主线程进行`accept`, 当有连接到来时.
将套接字描述符传给线程池中的对象, 使用全局共享资源即可.好处在于`accept`上是不需要加锁的.



上面介绍的就中网络编程模型, 是在`UNP`中所提到的九种模型, 在千禧年附近时还比较通用.不过如今已经有些不适应时代主流了.


下面我们介绍几种符合现今时代潮流的网络编程模型:
### Reactor (IO Multiplexing + Nonblock IO)

`Reactor`模型是当今Linux上最流行的网络编程模型.
`Reactor`是指通过一个或多个输入同时传递给服务处理器的服务请求的事件驱动处理模式。

![Reactor](http://www.qiniu.evilcrow.site/server_reactor1.png)

其中有两个关键的组件:
1. `Reactor`: 也叫事件分发器, 负责进行事件监听和分发,(也叫作`EventLoop`, 包含`Poller`,`Channel`)
2. `Handler`: 对于每一个分发的事件, 都有其对应的处理句柄

使用`Reactor`已经可以比较完善的处理`IO密集型`需求了.
在这一步基础上, 我们还可以配合线程池进行处理, 以应对`计算密集型`的任务

相比与传统的阻塞式IO, `Reactor`在IO复用模型, 线程池的线程资源复用上作出了改变

根据`Reactor`事件分发器和`threadpool`线程池的配置数量不同:
1. 单Reactor, 单线程
2. 单Reactor, 线程池
3. 主从Reactor, 线程池

#### 单Reactor, 单线程

优点: 简单易用, 没有多线程, 进程通信等全部在一个线程中完成
缺点: 不能重分使用多核CPU的性能, 且在处理事件时, 无法处理其他连接事件, 容易产生性能瓶颈

这种模型适合, 业务简单, 处理迅速的应用, 比如`Redis`, 要求业务时间复杂度是`O(1)`

#### 单Reactor, 线程池

相比与上一种, 这一种在线程模型上扩充, 使用了线程池进行任务的处理
线程池的使用在**计算密集型**的应用中尤为明显, 当然它还是不能增加并发量, 不能提高响应速度

优点: 充分使用多核CPU的性能
缺点: 业务处理的瓶颈解决, 只有一个`Reactor`进行处理连接, 性能瓶颈由线程转移到`Reactor`上
同时, 线程间同步, 进程间通信, 会成为编码的困难所在

不是太复杂的应用基本都可以适用此模型, 比如`SSDB`便是使用的`单Reactor + 线程模型`

#### 主从Reactor

上一种模型中, `Reactor`成为性能瓶颈, 于是便有了`主从Reactor`的模式

`主从Reactor`主要是将`Reactor`实现在多个线程中
`主Reactor`复杂进行连接, 监听`listenfd`, 连接成功后, 将连接转交给`子Reactor`, 仅处理该连接上的事件. 这样的做法, 可以将所有连接事件的压力缓解到多个线程, 分工明确

我们常见的很多著名项目都是基于此模型, 比如`Nginx`, `Java NIO Netty`, `Memcached`都是

总结一下, `Reactor`模式, 具有一下特点:
- 编程简单
- 响应快
- 有可扩展性 , 可根据应用类型和机器配置调控
- 复用性强, 与具体业务逻辑无关, 复用性强

### Proactor(IO Multiplexing + AsyncIO)

`Reactor`模型, 事件处于就绪态进行分发, 然后执行对应的操作. 所以`Reactor`属于同步非阻塞网络模型
`Proactor`模型, 事件完成后通知完成事件, 进一步执行后续操作, 所以`Proactor`是异步网络模型

![Proactor](http://www.qiniu.evilcrow.site/server_proactor.png)

两者的区别在于: **一个事件就绪进行通知, 一个是事件完成后进行事件完成通知**

`Proactor`与`Reactor`的区别在于:
1. 提前注册事件, 事件就绪时自行进行调用.
2. IO操作异步完成后, (由OS完成), 在回调事件处理

但是`Procator`宥明显缺点:
1. 编程复杂, 代码支零破碎
2. 内存使用, 必须一直保持, 一旦缓冲区丢失, 也就是数据丢失了.
3. 在`Linux`上目前还没有成熟的`异步IO`支持, `AIO`并不能够满足需求, (`Win`下的`IOCP`真正实现了异步IO)

因此, 在Linux下的编程模型, 基本都是`Reactor`模型为主

## Reactor模拟Proactor

很理所当然的, 我们知道`Proactor`是比`Reactor`多做了一件事: 将就绪事件进行处理, 如拷贝到缓冲区
那么, 我们也可以使用`Reactor`去干这部分的事, 以此来实现`Proactor`

思路是这样的:
1. 主线程正常`Reactor`, 获取就绪事件
2. 进行IO操作, 并将数据插入一个队列
3. 工作线程从队列中获取到数据, 就是`Proactor`模式了.
4. 同时工作线程注册写事件,
5. 主线程遇到写事件直接去写

亦即是说: 实际上, 还是`Reactor`, 不过是发生在主线程的, `Proactor`的实现是在工作线程中的
不过,这种模拟只能达到, 编程模式上的改变, 性能上不能获得明显提升, 因为没有异步IO的使用

换句话说: **异步IO提升效率, 同步IO符合思维逻辑**
只有用同步的方式去编写异步代码 (`corutine`)
没有使用异步的方式去写同步代码,性能上没有提升, 反而编写起来更为复杂.(`Boost.Asio`为了库跨平台)


