---
layout: post
comments: true
title: 第七章 进程控制(二)
date: September 22, 2017 5:04 PM
excerpt: 接上一章,在进行了进程最后部分的学习后,现将进程最后部分的知识进行总结学习同时会附上进行my_shell项目文档实现的经验
categories:
- Unix/Linux
tags:
- LinuxC
---

*接上一章,在进行了进程最后部分的学习后,现将进程最后部分的知识进行总结学习*

*同时会附上进行my_shell项目文档实现的经验*

_ _ _

**一,进程的其他操作**

本部分是一些进程的其他操作,虽然不如**进程操作**的部分重要,但是还是有很重要的意义

**1.获取进程ID**

进程ID,是进程的标识之一,进程ID(即PID)的重要性不言而喻

而且,**通过进程ID,方便对进程进行其他的操作**

``` cpp
#incldue<unistd.h>
#incldue<sys/types.h>
pid_t getpid(void)      //获取当前进程ID
pid_t getppid(void)     //获取当前进程父进程ID

```
另外还有获取进程实际用户,有效用户的函数,在之前已经提到过,此处不再赘述

**2.setuid与setgid**

看过<鸟哥的Linux私房菜>的同学,可能对这两个概念了解的比较好一点

简而言之,这两个函数,使用来处理程序中对用户权限的处理问题

``` cpp
#include<sys/types.h>
#include<unistd.h>
int setuid(uid_t uid)
itn setgid(gid_t gid)

```
其中需要谨慎处理的是,root权限的问题

因为,一旦用root权限去处理,则最后会将uid与gid进行修改,最后会导致失去root权限的问题出现

下面来看一个简单的例子

``` cpp
#include<sys/stat.h>
#include<unistd.h>
#include<stdio.h>
#include<fcntl.h>
#include<sys/types.h>
int main(int argc,char **argv)
{
    int fd;
    
    printf("Process Start!\n");
    printf("Process uid is %d,euid is %d\n",getuid(  ),geteuid(  ));

    fd = open("test.c",O_RDWR);
    if(fd == -1)
        printf("Error!");
    else
        printf("open Suessfully!\n");

    close(fd);
    
    return 0;
}
```

上面例子的结果
![](http://img.blog.csdn.net/20170731225312247?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
此即为文件权限的问题

**3.改变进程的优先级**

改变进程的优先级,是用来将常用,频繁的进程的优先级提高,换句话来说,就是提高计算机的效率

需要掌握的有三个函数

``` cpp
#incldue<unistd.h>
int nice(int increment);

#incldue<sys/resource.h>
int getpriority(int which,int who);
int setpriority(int which,int who,int prio);

```

具体示例,还请自行进行测试

_ _ _

**二,my_shell经验分享**

**进程的知识,暂时告一段落,在此处,我想要和大家分享的是my_shell项目实践经验**

我先放出,我的源码,以及项目文档

[传送门](https://github.com/Evil-crow/Linux_C/tree/master/Chapter_VII/Shell)

想要分享的经验如下:

1. 进行编码前,多思考,多考虑,尽量考虑周到一点

2. 多和别人交流,会有意想不到的收获

_ _ _
现在是技术上的分享

3. 关于守护进程,创建出一个脱离当前shell环境的进程,使用守护进程,算是后台进程的一种

4. 信号的实现,使用signal系列函数 ```signal(SIGCHLD,SIG_IGN)```

5. 经过我亲身经历,makefile真的很有用,用了都说好,而且还能装逼(滑稽-> <-)

6. PATH环境变量的设置,注意/etc/profile 与~/.bash_profile,的区别,并且学会如何修改环境变量

![](http://img.blog.csdn.net/20170731225325868?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



以上即为我的分享,有不足的地方还请大家指出,

下次会进行线程控制的分享(有机会一定补上文件操作的坑)
