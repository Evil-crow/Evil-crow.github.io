---
layout: post
title: 与网络编程有关的系统接口
date: March 14, 2019 1:33 PM
excerpt: 网络编程和系统编程是密不可分的, 我们需要掌握一些相关的系统编程接口
categories:
- Server
tags:
- NetworkProgramming
comments: true
---

与网络编程相关的常用系统接口有:`pipe(2)`, `dup簇`, `sendfile(2)`, `mmap(2)`, `splice(2)`, `tee(2)`, `fcbtl(2)`

下面一一介绍这些函数:

## pipe

```cpp
#include <unistd.h>
int pipe(int pipefd[2]);

#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <fcntl.h>              /* Obtain O_* constant definitions */
#include <unistd.h>
int pipe2(int pipefd[2], int flags);
```
使用`pipe(2)`用来创建进程间通信的管道.
**注意, 这只是一条管道, 管道有两端**`fd[0]`**仅能用于读**, `fd[1]`**仅能用于写**
如果要进行双向通信, 必须创建两条管道.

`flags`可选项为:`O_NONBLOCK`, `O_DIRECT`, `O_CLOEXEC`

默认通道的设置是**阻塞的**,**通道大小为65535**,这些都是可以通过`fcntl(2)`进行修改的

如果进行双向通信, 还需要两条管道, 未免太麻烦了. 我们可以使用socket的一个API

```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int socketpair(int domain, int type, int protocol, int sv[2]);
```

前面的参数和`socket(2)`相同, 不过严格限制允许`domain = AF_UNIX`, 因为管道就是进行本地通信的.
最后的`sv`数组保存了两个描述符, 这条管道, 两端都是可读可写的. 是真正意义上的双向管道

## dup簇

简单描述:用来进行文件描述符的复制
```cpp
#include <unistd.h>
int dup(int oldfd);
int dup2(int oldfd, int newfd);

#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <fcntl.h>              /* Obtain O_* constant definitions */
#include <unistd.h>

int dup3(int oldfd, int newfd, int flags);
```

基本上, 没什么多说的注意事项, 详情可以查看`man page`

## mmap与munmap

这两个API分别用于进行地址空间的映射.我们用`mmap`申请一段内存空间, 可以将其作为进程间通信的共享内存, 使用结束之后, `munmap`进行内存空间的释放即可

```cpp
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags,
       int fd, off_t offset);
int munmap(void *addr, size_t length);
```

参数解释:
- `addr`:可以指定开始的内存地址, 不指定则由系统分配
- `length`: 申请内存空间的大小
- `prot`: 设置的内存空间标志, 包含,可读, 可写, 可执行, 不能被访问等
- `flags`: 控制内存被修改后的行为
- `fd`: 是被映射文件的文件描述符,
- `offset`: 是文件的偏移量

## 零拷贝函数

在进行开发的时候, 我们经常需要进行内存的拷贝, 比如从内核态到用户态, 从用户缓冲区到TCP缓冲区
那么, 中间层的用户缓冲区, 就会有开销, 所以介绍下面几个零拷贝函数

### sendfile

`sendfile`实现了在文件描述符之间的数据拷贝, 直接在内核中操作, 避免了内核缓冲区到用户缓冲区之间的拷贝开销. 以此来提高效率

```cpp
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
参数简明易懂, 其中要提到的一个内容就是:
*在kernel 2.6.33之前, `out_fd`要求必须是网络套接字*, **目前, 已经支持任何类型的文件了**

### splice

同样又是一个零拷贝函数, 用于在两个文件描述符之间进行拷贝操作
```cpp
#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t *off_in, int fd_out,
               loff_t *off_out, size_t len, unsigned int flags);
```
参数仍然简明易懂, 具体可查看`man page`
不过注意的是: 在进行管道操作的时候, 注意偏移量的设置

### tee

`tee(2)`用于在管道之间进行零拷贝操作
```cpp
#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <fcntl.h>
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
```
参数类同`splice(2)`, 其他操作无甚区别.

那么, 我们来讨论一下这几个零拷贝函数 (`sendfile(2)`不在范围内)
讨论`splice`, `tee`, `vmsplice`
其中:
- `splice`: 将数据从缓冲区移动到任意文件描述符或设备.反之，或从一个缓冲区到另一个缓冲区。
- `tee`: 在缓冲区之间`"拷贝"`数据
- `vmsplice`: 从用户空间`"拷贝"`进入缓冲区

我们上面的`"拷贝"`是有引号的, 为什么这么讲呢? **因为实际上没有拷贝.**
内核通过实现管道缓冲区, 作为引用计数指针的集合, 是属于内核内存的. 内核的`"拷贝"`buffer中的指针,是通过创建新的指针实现的, 然后增加这个页的引用计数, 只有指针被拷贝了, 并非缓冲区中的页面被拷贝.

## fcntl

`fcntl(2)`是进行文件控制的函数, 可以实现各种功能. 比较杂, 和`ioctl(2)`有的一拼

基本功能如下:
- 复制文件描述符
- 获取和设置文件描述符标志
- 获取和设置文件描述符状态标志
- 管理信号
- 操作管道容量

之前经常使用`fcntl(2)`进行`O_NONBLOCK`标志的设置, 但是现在在我们拥有新的Syscall之后
如: `accept4`, `dup3`等添加了`flags`参数的标志,可以省去这一次系统调用

