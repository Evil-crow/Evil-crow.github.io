---
layout: post
comments: true
title: 第七篇 进程控制(一)
date: September 22, 2017 5:01 PM
excerpt: 经历了上一周悲剧的学习,这一周,我决定让自己充实起来,而进程控制(Peocess_Control)的学习尤为关键所以,下面即为我的收获,本章的学习分为两部分第一部分是进程的了解,以及相关操作第二部分就是进程的一些其他操作,以及项目的一些实现要点
categories:
- Unix/Linux
tags:
- LinuxC
---


*经历了上一周悲剧的学习,这一周,我决定让自己充实起来,*

*而进程控制(Peocess_Control)的学习尤为关键*

*所以,下面即为我的收获,本章的学习分为两部分*

*第一部分是进程的了解,以及相关操作*

*第二部分就是进程的一些其他操作,以及项目的一些实现要点*

_ _ _

##### 一,进程

- 进程的概念

了解什么是进程,进程的概念以及特点.

进程的概念总结后有以下几点:(进程与程序,线程的区别)

1. 进程是一个动态的实体,是操作系统分配资源的基本单位

2. 进程与程序的区别在于:

 程序只是代码块,而进程是将代码移至内存中,之后为其分配资源以及空间运行中的程序,

 可以这样理解进程,即特点之一是动态的

3. 进程与线程:

 为了使计算机能够在同一时间执行更多的任务,又在进程中划分了多个线程,

 线程是操作系统所能操作控制的最 小单位,

 进程分配有资源和内存,对于这些内容,线程不单独享用,多个线程共享这些资源以及内存,

 同一个进程可以创建和撤销多个线程,

 多个线程可以在进程中并行进行.

以上即为进程的基本概念.

其实我们可以从计算机的发展过程来看:

1.进程最早是分时操作系统(一个计算机史上的分支方向)的基本单位

2.后来是面向进程编程的操作系统 **(例如 UNIX, Linux Kernel < 2.6)**

3.直到现在,是以面向线程编程的操作系统.进程只是线程的容器,  **(现代大多数操作系统,Linux Kernel>2.6)**

进程(Peocess)一直是十分重要的概念,可以说进程是计算机对计算机资源的调度方式

这样才能使计算机能够执行任务.
_ _ _ 

- 关于进程的特点:

动态性:进程是动态产生的,也是动态消亡的

并发性:多个进程可以并行进行

独立性:单个进程可独立进行,系统为其分配资源,是计算机调度的基本单位

异步性:进程以不可预知的方向前进,速度,耗时,都是不可预知的
_ _ _
- 进程标识符(process ID)_PID

进程是计算机操作系统资源分配的基本单位,那么,

计算机中同时进行的可能会有成百上千的进程,

所以对进程最直观的区分就是进行编号,即进程ID Process_ID

**关于进程标识符:**

> 所有进程标识符都是惟一的

> 所有进程标识符非负

> 所有的PID,在进程开始时,为其分配PID,之后对其进行操作,进程结束后,PID回收

Linux操作系统中提供了用于获取PID的相关函数,如下:

头文件:
``` cpp
#include<unistd.h>
#include<sys/types.h>
```

|函数声明|功能|
|-------|----|
|pid_t getpid( )|获取到当前进程ID|
|pid_t getppid( )|获取当前进程父进程ID|
|Pid_t getuid( )|获取当前进程实际用户ID|
|pid_t geteuid( )|获取当前进程有效用户ID|
|pid_t getgid( )|获取当前进程实际用户组ID|
|pid_t getegid( )|获取当前进程有效用户组ID|


既然提到进程ID,就不得不进行细分:

1.RUID 实际用户ID:进程的执行者ID,仅只有root可以进行修改

2.EUID 有效用户ID:进程需要执行的用户ID,可以理解为权限ID,即该进程的拥有者

3.SUID 保留设置用户ID:进程切换其有效用户(EUID)时使用

对于这三种用户ID,有这些需要注意的点:

> 1.一般用户可以将EUID设置为RUID,SUID其一,root可以将EUID设置为任意合法的UID

> 2.当未设置SUID时,SUID = RUID,当设置SUID后,SUID = EUID

> root将S,E,RUID进行修改,修改他们的值.一般用户,仅当UID = R/SUID时,可将EUID置为UID,且不会修改值

- 进程的结构

之前上面也提到过,进程时动态的实体,既然是实体,那么他必然有其既定的数据结构

如图:
![](http://img.blog.csdn.net/20170727110851990?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

代码段:即是这个进程的代码

数据段:保存的是,进程中的全局变量,常量,静态变量

堆栈段:栈用于函数调用,其中保存着函数的参数,函数的内部定义的局部变量

**注意:这三部分实体构成了进程的结构,之后在进程操作中,要特别注意三个段数据的变化**

- 进程的状态

运行状态:R,**实际上,运行状态与就绪状态,统称可运行状态,R表示**

可中断的等待状态:S,**现在可中断的等待状态,称为可终端的睡眠状态,可以被其他程序唤醒**

不可中断的等待(睡眠)状态:D,不可被唤醒,直至等待事件完成

僵死状态:Z,已终止,描述符仍在,直至父进程调用wait后才能释放

停止状态:T,表示进程暂停,或者正在被信号跟踪(gdb调试)

同时使用后缀字符进行状态的补全:

**< :高优先级程序**

**N :低优先级进程**

**L :内存锁页,此页不可以被换出内存**

**s :该进程为会话首进程**

**l :多线程进程**

**+ :进程位于前台进程组**

既然上面提到了这么多的进程状态,我们就来看看怎么查看进程状态:

```
ps  查看进程的基础命令

常用选项:

-A,显示所有进程,

-u显示指定用户进程

-ef 显示进程信息,搭配命令行(可配合管道使用)

常用:

ps -aux 即可成功查看,当前计算机上所有进程

```

如图:

- 进程的内存映像

说了这么多,都是基于软件上的进程概念,那么,实质上进程是如何反映在内存中的呢?

一,将程序拷贝至内存中,使程序变成进程,之后,就是进程在内存中的映像

二,进程在内存中的映像

何为映像?

**我的理解就是,映像,即为进程在内存中的实际反映,即是,进程在内存中是如何编排的**

首先,来看两张图(图片引自网络)
![](http://img.blog.csdn.net/20170727110724278?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](http://img.blog.csdn.net/20170727110743139?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从内存的低地址开始,依次填入进程结构块中的内容

通过了解内存映像,可以加深对于进程概念的理解:

可执行程序没有堆栈,存储在硬盘中;进程存储在内存中,有堆栈段

可执行程序存储在硬盘,是静态的,不变的.而进程储存在内存中,是变化的,动态的.

_ _ _
##### 二,进程控制

了解了进程之后,进程控制则是重点内容

进程控制,主要有以下方面:

- 创建进程

- 创建守护进程(后台)

- 退出进程

- 执行新程序

- 等待进程结束

**1.创建进程**

进程很重要,那么重中之重就是进程的创建了,因为只有创建了进程,才能有后续的进程操作.

在UNIX/Linux中进程的创建只有两种类型:

> 由父进程创建出新的子进程

> 由操作系统内核创建出的一些特殊进程,比如,init进程,0,1号进程

而我们平时进行进程创建的多是第一种类型,即从父进程重新创建出新的子进程

来理解进程的生命过程:

**开始 -> 调用函数创建进程 -> 进程执行 -> 进程终止 -> 传递信息给父进程 -> 父进程进行资源的回收 -> 结束**

而进程在创建时,则是:

**开始->为进程分配惟一的PID->分配数据段,堆栈段**

首先介绍一个函数,fork( )函数,这是UNIX/Linux中特有的函数,不同于Win下的CreatProcess( )

``` cpp
#incldue<unistd.h>
#include<sys/types.h>
pid_t fork(void);

```

fork在英语中的语义是"分叉,分支",在程序设计里则是创建新进程的不二法宝.

fork( )一个新进程,主要有两点作用:

1.使用多进程执行一个程序(通常用处不大)

2.fork( )之后,立即调用exec函数族,执行另一个程序(常用法)

现在来理解一下fork( )的实际过程:

> 调用fork( )之后,操作系统立马新建一个进程在内存中,称为子进程.

> 子进程与父进程共享代码段,其数据段以及堆栈段为父进程的拷贝,即,调用fork( )之后,

> 创建了两个几乎完全相同的进程,而两个进程的关系,更像是兄弟关系.

fork进行进程的创建,当进程fork( )创建完后,

**fork( )这个特殊的函数会返回两个返回值**

来画一张图理解:

![](http://img.blog.csdn.net/20170727110815381?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如上图,进程fork后,新建了一个子进程,

两个进程并行运行,互相各走各的,

**所以希望大家注意一个误区,现在是两个进程在执行fork之后的相同代码**

**所以才会有不同输出结果! ! !**

看下面这段程序:

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
int main(int argc,char **argv)
{
   pid_t pid;
  /* printf("Fork start!");*/            /*子进程对缓冲区的数据实行了拷贝,虽然fork( )在printf之后*/
   printf("Fork start!\n");
   pid = fork( );

   switch(pid)
   {
      case 0:
         printf("I'm child process! my pid is %d,my father pid is%d\n",getpid( ),getppid( ));
         break;
      case -1:
         printf("Error!");
         break;
      default:
         sleep(1);
         printf("I'm parent process!my pid is %d,my parent pid is%d\n",getpid( ),getppid( ));
         break;
   }

   return 0;
}

```
![](http://img.blog.csdn.net/20170727110942485?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
由其进程ID,可以看出其父子进程关系,

比較有意思的是:

Fork start!,打印了一次

但是如果,是使用了上面的那條語句,則會打印两次Fork start!

这是为什么?

**这就要提到printf的机制了,他先将内容输出到缓冲区中,然后等待缓冲区刷新进行内容的输出**

**但是,如果缓冲区没有刷新,也就是说,子进程会将缓冲区拷贝,然后进行输出,在最后程序结束时**

可以总结一下,缓冲区刷新的条件:

1.缓冲区满溢出

2.\n ,\t,\v等强制刷新化冲区

3.程序结束时,不管是使用return还是exit,都会清理I/O输出,将缓冲区的内容输出,(默认return,接下来一个例子证明)

再来分析一下上面的程序:

> pid获取fork( )的返回值,fork( )之后是两个进程分别进行fork( )后面的代码

> swich语句中,

> 父进程获取到的pid是fork第一个返回值--子进程的PID,都是非0的

> 子进程获取到的pid是fork第二个返回值--0,因为子进程无子进程

> 如果fork( )失败,则会返回-1

fork的返回值很特殊,在进行程序设计时,需要明辨各自的返回值

那么,fork( )的实质清楚了,那么父子进程的运行情况又是什么样的呢?

看代码:

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
int main(int argc,char **argv)
{
   pid_t pid;
   int k;
   char *s;
  /* printf("Fork start!");*/   //同上例的缓冲区问题
   printf("Fork start!\n");
   pid = fork( );
   switch(pid)
   {
      case 0:
         s = "I'm child process! my pid is ,my father pid is\n";
         k = 3;
         break;
      case -1:
         printf("Error!");
         break;
      default:
         /*sleep(1);*/
         k = 5;
         s = "I'm parent process!my pid is ,my parent pid is\n";
         break;
   }
   while(k--)
   {
      printf("%s\n%d\n",s,k);
      sleep(1);
   }

   return 0;
}

```
结果:
![](http://img.blog.csdn.net/20170727111020554?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
其实但从上面结果来看,结果是不甚明显的,可以看到子父进程勉强是交替进行的,

如果把上面的sleep函数去掉,你会发现,其实是一个进程结束后,另一个进程才进行.

这一定就是对的吗?

事实上,在了解了操作系统的相关内容后,会发现,

**父子进程其实是杂乱无章的,而且是随机的,你所看到的规律是因为试验结果比较小导致的**

**真正决定子父进程顺序的其实是操作系统的调度算法,以及时间片的分配**

**进程具有异步性,反映出来就是父子进程是随机进行的**

**而且,需要给大家补充的一个概念,就是计算机同一时间只能处理单进程,多线程**

**之所以我们平时可以看到计算机同时进行多个任务,这是因为计算机切换进程的速度太快,约1/60s**

_ _ _

既然上面提到了,父子进程是随机的,主要依靠计算机CPU的调度算法,现在来了解两个特殊的进程:

**1.孤儿进程**

既然父子进程有异步性,不同时开始,不同时结束.那么,一旦父进程先于子进程结束,

则子进程称为**"孤儿进程"**

**因为此时该子进程失去了其父进程,而在UNIX类操作系统中,这样的事是不被允许的,一个子进程必须有父进程**

这个时候,特殊进程,PID == 1的进程,init进程就派上用场了,

**init进程会收养该子进程,成为该子进程的父进程,处理之后的一系列动作**

来看一段代码

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
int main(int argc,char **argv)
{
   pid_t pid;                       /*获取PID的参数*/
   printf("Fork start!\n");
   pid = fork( );                   /*创建新进程并获取其PID*/

   switch(pid)
   {
      case 0:
         sleep(1);          /* 此语句用来睡眠,使得子进程后于父进程结束,研究子进程pid的变化*/
         printf("I am child process my pid is %d my parent process pid is %d\n",getpid( ),getppid( ));
         break;
      case -1:
         printf("Fork Error!\n");
         break;
      default:
         printf("I am parent process,my pid is %d, my parent process pid ios %d\n",getpid( ),getppid( ));
   }

   return 0;
}

```
结果:
![](http://img.blog.csdn.net/20170727111042059?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
注意子进程的ppid(父进程ID),并非上面父进程ID,而是一个奇怪的数字1560,有时是1578,

**在/proc中可以看到,里面的编号其实都是PID,15xx,在进入目录,cat status后,会发现**

**都是名为systemd,1号也是systemd,可以说有多个init进程来接管孤儿进程**

你可能会疑惑,为什么对于孤儿进程必须要找一个人来收养,

那么,来看下面一个特殊的进程

**2,僵尸进程**

上面提到父进程先结束成为孤儿进程,那么子进程先结束便称作**僵尸进程**

僵尸进程,不要听上去就感觉耸人听闻,其实他更重要的是因为,进程处于"Z+",僵死状态

**现在来解释上面提出的一个问题:为什么一定要找出一个进程来收养孤儿进程**

**因为,在UNIX类操作系统中,一个进程消亡,并非是真正的死亡,而是会留下一个称为"僵尸进程"的数据结构**

**因为,UNIX类操作系统使用此机制,可以避免因为异步的特性,子进程先结束,而是父进程失去子进程信息的情况**

那么,这也就带来了一个问题,**有大量地僵尸进程,则会占用进程标识符PID,最终导致资源泄漏**

来看一段代码:

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
#include<stdlib.h>
int main(void)
{
   pid_t pid;
   int i = 5;

   pid = fork( );
   switch(pid)
   {
      case 0:
         printf("I am child process my pid is %d,i = %d\n",getpid( ),i++);
         _exit(0);
         break;
      case -1:
         printf("Fork Error!");
         break;
      default:
         sleep(5);
         printf("I am parent process my pid is %d,i = %d\n",getpid( ),i++);

   }

   return 0;
}

```

结果:
![](http://img.blog.csdn.net/20170727111104442?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
其中Z+即为子进程,僵尸进程

至于为什么需要进程来收养孤儿进程 **就是防止不可回收的僵尸进程**

僵尸进程大量存在,会严重危害计算机的工作效率

来看看集中解决僵尸进程的办法:

1> 父进程调用wait或者waitpid函数等待子进程的结束

2> 使用信号SIGCHLD,用忽略信号,使得父进程忽略信号,子进程直接移交内核处理

3> 小技巧:kill "僵尸爸爸",因为一般情况下.僵尸进程很难被杀死

4> 小技巧:连续fork( )两次,并杀死第一次的子进程,之后使用被收养的孤儿进程exec族函数调用新程序

**这里,需要给大家明确一个概念:僵尸进程,是很正常的一种情况**

**是进程结束的必经过程,不过有的进程在进行合理处理后,僵尸进程的时间很短暂,但还是经历了Z+状态**

**而且,不要将僵尸进程理解成有害而无利的,僵尸进程是因为父进程可能需要获取子进程信息而存在的**

**在最后,切记一点,在进行僵尸进程的测试时,ps -aux命令需要在另一个终端中使用(个中原因大家自行体会)**
_ _ _

**vfork( )创建新进程**

上面的内容都是以fork( )创建新进程为主,那么vfork( )有什么作用呢?

**vfork( )也是创建新进程的函数,同样也是调用一次,有两个返回值**

**vfork( )和fork( )的区别在于:**

**1.vfork( )共享代码段,和数据空间,两个进程是共享的,而fork( )只有代码段共享,其他部分都是分离的**

**2.vfork( )调用后,父进程会等待子进程结束才继续进行,相当于使用了wait/waitpid的fork( )函数**

**由上面的1.即可知,子进程对于一些全局变量的操作在父进程中也是可见的,因为他们共享,而非拷贝**

**如果在子进程中依赖父进程的行为,则会造成死锁,(互相等待对方进程的完成)**

**最后,十分重要的一点,只有调用了exec族函数/exit,才能由子进程回到父进程**

来通过代码看结果:

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
#include<stdlib.h>
int b = 5;
int main(int argc, char **argv)
{
   pid_t pid;
   int a = 3,i;
   pid = vfork( );                                  /*创建子进程*/
   /*pid = fork( );*/
   switch(pid)                                          /*进行pid判断*/
   {
      case 0:
         i = 3;
         while(i--)
         {
            a++;
            b++;
            sleep(1);
            printf("a = %d,b = %d\n",a,b);
         }
         //_exit(0);                                        /*注意此处应该如何对子进程进行操作*/
         //return;
      case -1:
         printf("Error!");
         break;
      default:
         i = 5;
         while(i--)
         {
            a++;
            b++;
            sleep(1);
            printf("a = %d,b = %d\n",a,b);
         }
         break;
   }

```

在注释掉case 0结束时的两句话时,结果是这样
![](http://img.blog.csdn.net/20170727111134782?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在使用return,结果也这样(顺便可以了解,函数结束时若不作处理,默认是return,返回)
![](http://img.blog.csdn.net/20170727111218674?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**这就要讲到,return与exit机制的区别了**

**return是关键字,而exit是函数**

**return是函数结尾,用于返回值使用.exit是函数中,用于结束进程使用**

**尤其是在vfork( )函数中,因为父子进程共享空间**

**那么,如果使用return,在子进程结束时,会弹栈,破坏公共的堆栈段,等父进程结束时,就会出现错误**

**可能会返回诡异的栈值,才会出现奇怪的结果,如图一**

**而使用exit就不会出现这样的结果,exit函数进行进程的结束,并不会破坏公共堆栈段**

**这就是为什么要使用exit/exec函数族的原因了.**

另外,我们再来谈谈exit这个函数

**这是个神奇的函数,exit为libc函数,_exit为系统调用Systemcall,exit函数,是封装_exit系统调用而来的**

**而这两个最大的区别就是,exit会清理I/O输出,即会将所有缓冲区的内容输出,或者写入文件**

**在main函数中exit(0) ~~ return 0**

_ _ _

上面的内容即为创建进程的主要内容了,

**在这里我想要分享的是,fork( )函数,vfork( )函数其重要作用都是为了新建子进程后调用exec族函数**

**执行其他程序的,我们这里是为了了解函数特性才这么做的**

**虽然vfork( )也是比较好用的,但是现行标准是不建议使用vfork( )的**

**因为vfork( )共享数据段,堆栈段.操作十分干危险,而且在子进程中影响父进程,在实际开发中是十分危险的**

**如果,vfork( )之后不是立刻进行exec调用,就要十分小心,谨慎判断**

**但是现在,fork(　）实现了优化，实现了写时拷贝，则使得fork( )之后exec的代价十分小了**

**所以,现在的建议还是,使用fork( )为好**

_ _ _

**2.创建守护进程**

何为守护进程?简单点讲,就是后台进程,不会因为终端键入的信息而响应,也不会将错误信息发送到屏幕上,

此处,我们对守护进程并不深入展开,仅进行守护进程的创建步骤解析:

1.fork( )创建新进程,并结束父进程,交由init接管

2.setsid( )创建新会话,并使该子进程成为会话组长

3.再次fork( ),创建新的子进程,并且结束父进程

4.关闭不必要的文件描述符,

5.设置掩码为0

6.处理信号

通过以上的步骤即可建立一个后台守护进程.

结果如图:
![](http://img.blog.csdn.net/20170727111155217?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
可以看到,没有反应,表面上,然后可以发现后台有一个进程在执行

_ _ _

**3.进程的退出**

进程的退出,Linux系统中表示进程的结束,退出的方法分为正常退出和异常退出,主要有以下的方法:

(1)正常退出:

return 进行返回(main中结束进程)

调用exit,_exit,_Exit函数

最后一个线程从启动例程返回

最后一个线程调用pthrad_exit

(2)异常退出:

调用abort函数

接受到信号并终止

最后一个线程对取消请求响应

其中重点注意return,exit的区别,exit,_exit的特色,以上已经讲过,在此不表

_ _ _

**4.执行新程序**

上面提到过fork( ),vfork(　)，实际的重要作用,是创建一个新进程后,立刻进行新程序的调用

**而提到新程序的执行,不得不提到的便是exec族函数**

**通过exec族函数的调用,可以用新的程序来替代当前的进程映像**

**注意:调用exec族函数并没有新生成进程,调用的同时,原进程"死亡",其代码段被替换**

**为其分配新的数据段和堆栈段,惟一相同的就只有PID了,可以说还是同一个进程,但进行的已经是其他程序了**

因为exec族函数要调用环境变量,先来了解一下环境变量

**通俗点讲,环境变量path,就是在使用程序时,没有告诉其完整路径,系统除了在本目录下寻找以外**

**还应该去到指定的path路径下去寻找**

使用 ```env```命令可以查看当前的环境变量

**显示环境变量有两种方式,看代码**

**第一种,使用在别处定义的全局变量environ**

``` cpp
#include<stdio.h>
#include<malloc.h>

extern char **environ;                  /*在别处定义的系统变量(预定义)*/

int main(int argc,char **argv)
{
   int i;

   for(i = 0;i < argc;i++)
   {
      printf("argv[%d] is %s\n",i,argv[i]);
   }

   for(i = 0;i < argc;i++)
   {
      printf("%s\n",environ[i]);
   }

   return 0;
}

```

**第二种,使用main函数的完整形式,并打印其值envp**

``` cpp
#include<stdio.h>
#include<malloc.h>
int main(int argc,char **argv,char **envp)
{
   int i;

   for(i = 0;i < argc;i++)
   {
      printf("argv[%d] is %s\n",i,argv[i]);
   }

   for(i = 0;i < argc;i++)
   {
      printf("%s\n",envp[i]);
   }

   return 0;
}

```

使用上面的两种方法都可以获取到环境变量的值.
![](http://img.blog.csdn.net/20170727111656922?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
exec族函数中,execve为惟一的系统调用,其他的函数都会调用此系统调用

![](http://img.blog.csdn.net/20170727111433999?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](http://img.blog.csdn.net/20170727111445309?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**exec族函数的详细用法,查看man手册**

来看一段exec演示代码

``` cpp
//调用的新程序
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
int main(void)
{
   printf("HaHa I'm the program to replace the process!\n");
   printf("My pid is %d,my parent pid is %d\n",getpid( ),getppid( ));
   return 0;
}
```

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
#include<stdlib.h>
int main(int argc,char **argv,char **envi)
{
   pid_t pid;
   printf("Exec start!\n");
   pid = fork( );
   if(pid == -1)
   {
      printf("Error!");
      exit(0);
   }
   else if(pid == 0)
   {
      printf("I am child process !\n");
      printf("my pid is %d,my parent pid is %d\n",getpid( ),getppid( ));
      execve("program",argv,envi);
      printf("This sentence won't be print!\n");
      _exit(0);
   }
   else
   {
      sleep(2);
      printf("This is parent process!\n");
   }

   return 0;
}
```

结果会发现,调用exec族函数后,进程的PID是不会改变的,
![](http://img.blog.csdn.net/20170727111555352?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
其中还有许多项目是保留的:

1.当前工作目录

2.根目录

3.使用的屏蔽字

4.控制终端

5.文件锁

_ _ _

**5.等待进程结束**

前面提到过,父进程需要等待子进程的结束,可以避免僵尸进程

那么,如何等待进程的结束,调用wait,waitpid函数

``` cpp
#incldue<sys/types.h>
#include<sys/wait.h>
pid_t wait(int *statloc)
pid_t waitpid(pid_t pid,int *statloc,int options);

```
**wait函数会返回等待进程的PID,如果statloc指向不为空,会将退出码储存在statloc指向的变量中**

**在这里需要注意的是,退出码是一个字段,并非是单个数字,可以用宏去取出退出码中的信息**

举个例子:

**WIFEXITED(&statloc)**即可取出其中的信息

关于wait函数的宏,详情请查找man手册,

现在介绍waitpid函数,其中pid表示目标子进程的PID,statloc意义相同,options是可以添加的选项

**可以看出,waitpid函数yuwait的一个重要区别就是,wait只等待第一个结束的子进程**

**而waitpid函数,可以等待指定的子进程结束,而且waitpid提供了wait的非阻塞版本**

**即,在调用了waitpid后,使父进程不被挂起,而立刻返回,提供了WNOHANG这样的一个选项**

下面是一个演示的例子

``` cpp
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>
#include<sys/stat.h>
#include<stdlib.h>
#include<sys/wait.h>
int main(int argc,char **argv)
{
   int status;                              /*用来记录wait返回值*/
   int pid;                                 /*记录fork的返回值*/
   int exit_code,k;                           /*设置退出码,用于获取结束信息*/

   printf("fork start!\n");
   pid = fork( );                      /*新进程已经建立,父子进程并行,或者说是操作系统对进程的快速切换*/
   
   switch(pid)
   {
      case 0:
         k = 4;                             /*之后子进程循环操作,使父进程阻塞的依据*/
         exit_code = 45;                    /*子进程设置退出码,之后可以在父进程中获取*/
         break;
      case -1:
         printf("Error!");
         break;
      default:
         exit_code = 0;
         break;
   }

   if(pid [^]!= 0)
   {
      int status;                          /*此即为wait函数中的statloc变量,用于获取退出码*/
      pid_t  child_pid;                   /*wait函数的返回值,用于取得结束的子进程的pid*/

      child_pid = wait(&status);           /*获取到退出码(退出码中储存着退出时的信息,是一个字段,可用宏取出其中信息)以及结束的子进程的pid*/

      printf("Child process has exited!It's pid is %d\n",child_pid);
      if(WIFEXITED(status))
      {
         printf("Child process has exited,It's exit_code is %d\n",WEXITSTATUS(status));   //WEXITSTTUS宏用来获取exit函数低8位
      }
      else
         printf("Child process exit abnormally\n");
   }

   while(k--)
   {
      sleep(2);
      printf("Child process is running!\n");
   }
   exit(exit_code);
}

```

结果:
![](http://img.blog.csdn.net/20170727111339029?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
可以看到父进程处于阻塞状态,即在等待子进程的结束

_ _ _

以上即为基本的进程操作,如果代码有问题,可以到我的Github上来进行查看

++[传送门](https://github.com/Evil-crow/Linux_C/tree/master/Chapter_VII)++

下次会谈谈进程控制中的其他操作,诸如改变进程的优先级,用户的权限操作之类的,

如果本篇Blog有问题,可以在评论区中提出

July 27, 2017 11:00 AM
