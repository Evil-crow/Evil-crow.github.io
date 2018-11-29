---
layout: post
comments: true
title: 第八章 线程控制(一)
date: September 22, 2017 5:06 PM
description: 上一次的内容是进程控制,到了这一章则是比进程用处更广泛的线程了学好线程,可是有很大的作用
categories:
- Unix/Linux
tags:
- LinuxC
---

*上一次的内容是进程控制,到了这一章则是比进程用处更广泛的线程了*

*学好线程,可是有很大的作用呢!*

\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

**一,线程的概念**

在进行线程的学习时,首先来了解一下概念:

**线程是进程中的一个实体,自己不拥有资源,(除了运行中的必不可少的资源,如堆栈段,寄存器)**
**一个线程可以创建/撤销其他的线程,同一个进程中的多个线程可以并发执行**,由于线程之间的相互制约
有间断性{就绪,阻塞,运行}
**每一个进程至少存在一个线程,如果只有一个线程,那么这个线程 = 进程 = 程序自身**

再用用户看来,多个线程同时进行,实质上线程还也是类似于进程一样在进行调度运行,叫做**并发**

但是,要注意的一点是:**在多核CPU的现世代,多个线程可以在不同的CPU上工作,即真正意义上的并行**

_ _ _

为什么要有多线程,因为它比多进程有这些优点:

1. 多进程是独立的地址空间(vfork除外)

 反观多线程是独立的地址空间,**同一进程内的线程,共享此进程的内存空间**
 
 **所以,多线程更具有时间优势**

2. 系统调度方面

 在系统调度方面,**因为线程共享内存空间,所以在线程调度切换上更具有优势,内部直接进行通信**
 
 而进程之间的系统调度,**则需要信号机制调度,必经过操作系统,时间上更麻烦**
 
3. 另外的一些优势

 使用线程,可以提高响应速度,可以提高多处理器效率,改善程序结构

在我自己的理解看来,

**线程也没有多神秘,其实就是在进程中继续划分的更小的计算机工作单位,可以更好地提高效率,节省时间**

上面提到了,同一进程内的线程线程共享进程的地址空间,但是,线程也有其私有数据(区别于之后的**"私有数据"**)

> 线程号,ThreadID

> 寄存器 [程序计数器,堆栈指针]

> 堆栈

> 信号掩码

> 优先级

> 线程的私有存储空间

Linux系统下,不同于进程,线程的实现方式基于POSIX多线程标准

所有线程的操作,都是基于POSIX基准的接口函数

所以,**在进行编译时,要链接动态库 -lpthread**

下面说一些线程其他事情吧:

_ _ _

1. 为什么会有线程出现?

 上世纪60年代,出现了进程的概念,方便工程师进行程序的设计,

 但是,随着科学日益进步,工程量的增加,进程已经明显不能满足需求了

 主要问题体现在两点上:

 - 进程较大的时 + 空开销(时间,空间的开销都大)

 - SMP,对称多处理机的出现,是同时运行几个程序成为了可能,但是使用进程的开销会很大

 就这样,在需求不段增长的时候,80年代,线程就出生了.

2. 线程有几种调度方式?

 线程进行调度分为三种:

 **<1>. 操作系统内核线程 e.g.Win32线程**

 **<2>. 用户线程 e.g.POSIX Thread**

 **<3>. 内核与用户线程混合调度 e.g.Win7 线程**

3. 线程的属性

 最早在Linux环境下,并没有真正的实现线程,所使用的名为"轻量进程",实则用进程来实现线程功能
 
 直至后来,才再真正意义上实现了线程
 
 **线程 = 程序(代码) + 数据 +　TCB(类似于PCB)**
 
 而其动态特性,则由TCB进行描述:
 
 **线程状态 + 线程不运行时资源问题 + 执行堆栈 + 线程局部变量主存区 + 访问同一进程主存与其他资源**
 
 **最后,用一句话来描述线程与进程的关系 : 进程是线程的容器,线程是进程执行程序真正实体**

\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

**二,线程操作**

**1.创建线程**

``` cpp
#incldue<pthread.h>

int pthread_creat(pthred_t *thread,pthread_attr_t *attr,
                  void *(*start_routine)(void *), void arg)
```
pthread_creat,线程创建函数

其作用是:**创建线程号为thread,线程属性为attr,执行参数为arg的start_routine函数的线程**

**创建一个新的线程后,线程也去执行新的程序,类似于进程的exec系函数,但是在内存空间上分配有不同之处**

**新创建的线程去运行指针指向的函数,而原线程继续执行接下来的操作**

再来看几个函数:

|函数  |  说明  |
|-----|-------|
|pthread_t pthread_self(void)|类似于getpid(),获取线程自身线程ID|
|int pthread_equal(pthread_t thread1,pthread_t thread2)|判断两个进程是否为同一进程|
|int pthread_once(pthread_once_t *once_control,void(*int_routine)(void))|保证该函数仅执行一次|

下面来看看如何创建进程:

``` cpp
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

int *get_thid(void);

int main(int argc,char **argv)
{
    pthread_t thid;                     //声明进程ID变量

    printf("parent pthread is me,my thid is %lu\n",pthread_self( ));
    if(pthread_create(&thid,NULL,(void *)get_thid,NULL) != 0)
    {
        printf("Error!\n");                 //调用函数进行进程的创建
        return 0;
    }
    sleep(1);
    return 0;
}
int *get_thid(void)                      //创建进程时,被调用的函数
{
    pthread_t thid;

    thid = pthread_self( );
    if(thid < 0)
    {
        printf("Error!\n");
        exit(0);
    }
    printf("I'm child pthread,my thid is %lu\n",thid);
    return NULL;
}

```

如图所示,即为结果
![](http://img.blog.csdn.net/20170807192925226?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
_ _ _

**2.线程属性**

在线程的创建函数中一个参数是attr,其类型为pthread_attr_t,此结构体定义如下:

``` cpp
typedef struct {
    int                      detachstate;
	int                      schedpolicy;
	struct sched_param       schedparam;
	int                      inheritsched;
	int                      scope;
	size_t                   guardsize;
	int                      stackaddr_set;
	void *                   stackattr;
	size_t                   stacksize;
} pthread_attr_t;

```
以上即为attr,线程属性的定义,类似于文件描述符FILE的定义,

线程的属性设置又有重要的作用,因为此处目前没有太多作用,不予深究,有兴趣的可自行查阅man手册

_ _ _
**3.线程的终止**

在Linux环境下,有两种方式实现线程的终止

- 调用return函数,实现线程终止

- 使用POSIX标准的接口API,pthread_exit函数

**这两个函数主要的区别之处在于在主线程中调用的区别:**

**在主线程中调用return/exit,会使主线程结束,进而整个线程结束,全部线程消亡**

**如果是调用pthread_exit( )函数,则主线程消亡后,其他线程并不会受到影响,知道所有线程结束,进程才会结束**

_ _ _

在线程的终止时,另外一个重要的问题就是关于资源的释放问题:

**特别是一些临界资源,临界资源在同一时间只能被其中一个线程所使用,如若被多个线程使用,则会导致资源混乱**

而如果,临界资源给一个线程所使用,的那是线程退出时没有释放临界资源,

**则其他线程会一直认为该临界资源还在被其他线程所占用,就会导致死锁问题的出现**

**死锁问题的出现,在程序设计的过程中,往往是灾难性的**

所以为了妥善处理线程结束时,临界资源的释放问题,Linux系统提供了一对函数:

``` cpp
#include<pthread.h>
#define pthread_cleanup_push(routine ,arg) \
{
	struct _pthread_cleanup_buffer buffer; \
	      _pthread_cleanup_push(&buffer,(routine),(arg));
#dedine pthread_cleanup_pop  \
         _pthread_clean_pop(&buffer,(exeute));
}

```
上面的两个函数,pthread_cleanup_pop( )与pthread_cleanup_push( )是要配合起来使用的

\*cleanup_push( )用来在线程提前结束时清理函数,

而,\*cleanup_pop( )则是在线程正常结束时,用来清理*cleanup_push( )函数的

_ _ _

在释放资源以外,另一个需要谨慎处理的就是资源的同步问题了,

一般情况下,进程中各个线程的运行是相互独立的,线程的终止并不会相互通知,终止的线程资源仍归线程独有

**所以资源的同步十分重要,同进程中的wait函数,在线程中所使用的是pthread_join( )函数**

``` cpp
#include<pthred.h>
void phread_exit(void *retval);
int pthread_join(pthread_t thid,void *thread_return);
int pthread_detach(pthread_t thid);

```

函数pthread_join用来使调用者挂起等待thid线程的结束

要注意的点有:

**1.一个线程只能被另一个线程所等待,并且被等待的线程必须处于可join的状态,即它不能被设定为DETACHED**

** 处于DETACHED状态的线程是指内核不关心线程返回值,线程结束后,内核自动回收的分离模式**

** 所以,为了防止内存泄漏,并且完成线程同步,所有的线程结束时,都要设定为DETACHED或者被join( )等待**

**2.一个线程只能被一个线程等待,若被多个线程等待,其中一个线程恢复恢复就绪状态后,其他线程便进入了死锁**

``` cpp
#include<stdio.h>

#include<stdlib.h>

#include<pthread.h>

#include<unistd.h>

void test(void);

int main(int argc,char **argv)
{
    pthread_t thid;
    int status;

    pthread_create(&thid,NULL,(void *)test,NULL);
    pthread_join(thid,(void *)&status);                                     //使主线程进行阻塞,等待子线程结束
    printf("I  (%lu)  have waited for a long time  %d",pthread_self( ) ,status);        

    return 0;
}

void test(void)
{
    printf("I am for test !\n");
    sleep(20);                                                  //用sleep来延时函数
    printf("I have achieved!\n");

    pthread_exit(0) ;
}
```

以上结果即可看出,调用函数对目标函数完成了挂起等待.
![](http://img.blog.csdn.net/20170807192942808?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
_ _ _

**4.私有数据**

**区别于之前提到的私有数据,此处私有数据指的是多个线程中操作不同的数据,并非是..**

**在这里举一个特殊的的例子:errno全局变量**

**理论上errno应该是任何线程都够访问的全局变量.但是如若errno中保存的值还没有被使用**

**便被其他线程更改了其中的值,同样也会影响使用**

**像这种全局变量,即是我们此处要讨论的私有数据,即都能访问的全局变量,但是在各个线程中又是不一样的值**

私有数据的实现方式借用了:**一键多值**

此处对这个键可以理解为:**一个数据管理器,在各个线程中,调用时,键会告诉在此线程中应该使用什么值**

``` cpp
#include<pthread.h>

int pthread_key_creat (pthread_key_t *key,void (*destr funcation) (void *));

int pthread_setspecific (pthred_key_t *key,const void *pointer);

void *pthread_getspecific (pthread_key_t key);

int pthread_key_delete (pthread_key_t key);
```

上面的函数,

**Creat函数是用来创建键的,**

**setspecific函数用来将线程的私有数据与键绑定,在线程自身中调用**

**getspecific函数用来获取键值中绑定的私有数据**

**delete函数用来销毁键**

有以下要注意的点:

1.在使用pthread_key_creat函数时,

切记只能初始化一次如果一个键值被创建了两次,会覆盖

建议:**在创建键时,可以在main函数中创建一次,或者使用prhread_once函数只进行一次创建**

创建新键时,每个私有数据的地址为NULL.

2.在pthread_key_creat函数中,

使用了析构函数.所谓析构函数指的是用来在键值使用完成之后

清除并释放与键值绑定的私有数据所占的内存空间,

**键值对与私有数据所占用的并不是相同的数据空间,所以要分开进行释放**

**一旦在键值对释放时,未释放私有数据所占据的空间,则会导致内存泄漏,灾难性的后果**

**所以调用析构函数有其一定的必要性,当为NULL,会调用内核自身的清理函数**

一般情况下,**线程调用malloc为私有数据分配内存空间**

3.在pthread_delete函数中

**pthread_delete函数是用来取消键与私有数据间关联的函数**

调用pthread_delete函数并不会影响正在使用的私有数据与键值,但是容易造成内存泄漏

**最后总结,不同的线程对私有数据的访问对彼此之间是不可见的,操作互不影响,**

**即键同名且全局但访问内存空间不同**

**可以将key理解为一个数据管理员**

来看一个示例

``` cpp
#include<unistd.h>
#include<stdio.h>
#include<pthread.h>
#include<string.h>

pthread_key_t key;                //定义全局变量库--键

void *thread1(void *arg);          //线程1

void *thread2(void *arg);          //线程2

int main(void)
{
    pthread_t tid;                 //线程ID

    printf("main thread begins running!\n");
    pthread_key_create(&key,NULL);                        //参数为,键地址,以及析构函数(用于私有数据的内存清理),如果为NULL,则调用系统的清理函数
    pthread_create(&tid,NULL,thread1,NULL);               //四个参数依次是线程ID,线程属性,调用函数,函数参数
    sleep(10);                                            //睡眠以使主线程等待
    pthread_key_delete(key);                              //销毁键,私有数据的销毁必须在其之前,不然会内存泄漏
    printf("mian pthread ends \n");

    return 0;
}

void *thread1(void *arg)
{
    int tsd = 5;                                          //pthread中的私有数据
    pthread_t thid_1;                                     //分配新的线程号

    printf("pthread 11  %lu is running!\n",pthread_self(  ));
    pthread_setspecific(key,(void *)tsd);                        //使键与私有数据绑定
    pthread_create(&thid_1,NULL,thread2,NULL);            //创建新线程
    printf("thread1 %lu ends,pthread's tsd is %d\n",pthread_self(  ),pthread_getspecific(key));
    sleep(5);                                            //睡眠以等待新线程结束

}

void *thread2(void *arg)
{
    int tsd = 0;

    printf("pthread 22 %u is running\n",pthread_self(  ));
    pthread_setspecific(key,(void *)tsd);                       //绑定键值与私有数据
    printf("Thread %lu ends,thread's tsd is %d\n",pthread_self(  ),pthread_getspecific(key));

}

/* 对此段代码,其中需要注意的地方是,
 *
 * 一,关于Thid的问题,使用%d,整型根本保存不下线程ID,必须使用%u,不然会出现溢出
 *
 * 二,即是任意类型指针的问题,(void *)可以指向任何类型的数据,但是会出现警告
 *
 * 而在网路上的解法都是直接进行取地址去获取地址,并传参
 *
 * 至于细节,还需要再琢磨
 *
 */
 
```
![](http://img.blog.csdn.net/20170807192959580?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

本次线程控制总结至此,下次将会是线程同步的内容,十分重要!!!

另外还有小实验的总结,十分有用!

August 7, 2017 7:24 PM
