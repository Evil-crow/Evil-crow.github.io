---
layout: post
title: my_shell项目文档
date: September 22, 2017 5:21 PM
excerpt: my_shell 项目文档
categories:
- Documents
tags:
toc: true
comments: true
---

在**进程**的相关知识学习完了之后,按小组要求实现了my_shell

**一个类似与bash shell的命令分析器**

本篇即是我自己实现的my_shell的项目文档

my_shell的源码,已上传至Github,可以[点击这里](https://github.com/Evil-crow/Linux_C/tree/master/Chapter_VII/Shell)
_ _ _

本篇项目文档分为五个部分,可通过点击以下标签进行访问

* [一,项目需求](#1)

* [二,设计思路](#2)

* [三,实现要点](#3)

* [四,项目演示](#4)

* [五,存在问题](#5)

* [六,设计感想](#6)

_ _ _

#### 一.项目需求

**1.支持输入输出重定向(< > >>)**

**2.支持管道(|)**

**3.支持shell内置命令(实现cd即可)**

***4.支持后台运行程序(&)**

***5.支持tab补全目录文件以及历史记录上下翻(提示:使用readline库)**

***6.ctrl+c不能中断程序(提示:使用信号)**

***7.设置环境变量，使程序可以像bash，zsh这种shell一样运行**

_ _ _

#### 二,设计思路

本项目的主题思路是**系统编程+进程操作**

通过大量的系统编程来实现系统功能,

在加上进程操作来实现其他的功能

1.[main.c](https://github.com/Evil-crow/Linux_C/blob/master/Chapter_VII/Shell/main.c)

![](http://img.blog.csdn.net/20170731040746556?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2.[print_shell.c](https://github.com/Evil-crow/Linux_C/blob/master/Chapter_VII/Shell/print_shell.c)

![](http://img.blog.csdn.net/20170731040800867?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3.[get_command.c](https://github.com/Evil-crow/Linux_C/blob/master/Chapter_VII/Shell/get_command.c)

![](http://img.blog.csdn.net/20170731040850458?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

4.[explain_command.c](https://github.com/Evil-crow/Linux_C/blob/master/Chapter_VII/Shell/explain_command.c)

![](http://img.blog.csdn.net/20170731040911331?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

5.[go_command.c](https://github.com/Evil-crow/Linux_C/blob/master/Chapter_VII/Shell/go_command.c)

![](http://img.blog.csdn.net/20170731040924292?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


6.[find_command](https://github.com/Evil-crow/Linux_C/blob/master/Chapter_VII/Shell/find_command.c)

![](http://img.blog.csdn.net/20170731041554550?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

*以上的思维导图可以较好的反映功能的实现思路*

_ _ _
#### 三,实现细节

1. main.c的实现其实就是按照思路进行设计,将下面的函数有效的组织在了一起

2. 在print_shell的部分里,主要的环节就是打印shell提示符

 为了追求尽可能的真实,我就做了这样的设计

 **"[用户名@主机名 目录名] "**

 其中主要的设计在目录名的打印上,先使用getcwd系统调用获取绝对路径,再进行字符串操作,获取当前目录

 **注意其中最重要的是,根目录/的目录名打印需要特殊处理**

 代码实现:

 ```
 if(getcwd(dir,100) == NULL)
 {
     printf("Error!\n");
     exit(0);
 }
 int i,j;
 for(i = 0;dir[i] != '\0';i++)
 {
     if(dir[i] == '/')
     j = i+1;
 }
 if(dir[j+1] == '\0')
 {
     printf("[Crow@Shell /]Σ ");
     return ;
 }
 char Dir[30];
 for(i = 0;i < 100;i++,j++)
 {
     Dir[i] = dir[j];
 }
 Dir[i+1] = '\0';

 printf("[Crow@Shell %s]Σ ",Dir);

 ```
3. get_command的功能是进行命令的获取,并且保证其以'\n'结尾

 ```
 i = 0;
 ch = getchar( );
 while(ch != '\n')
 {
     str[i++] = ch;
     ch = getchar( );
 }

 if(i > len)
 {
     printf("Command isn't allowed!\n");
     exit(0);
 }
 if(strcmp(str,"^C") == 0)
 {
     exit(0);
 }

 str[i++] = '\n';
 str[i++] = '\0';

 ```
4. expian_command是为执行命令进行准备,对命令参数进行划分获取

 下面这段代码很好的演示了使用交叉指针进行移动并获取参数,存入字符串的方法

 ```
 q = p;
 n = 0;
 while(1)
 {
   if(*q != '\n' && *q != ' ')
   {
      q++;
      n++;
   }
   else
   break;
 }
 strncpy(parameter[count],p,n+1);
 parameter[count][n] = '\0';
 count++;
 p = q;

 ```

5. go_command是整个程序的核心,其中进行了参数的判断,并使用switch函数进行功能选取

 尤其好用的操作就是在处理父子进程的问题上

 ```
 if(background == 1)
 {
    printf("processid is %d\n",pid);
    exit(0);
 }
 if(waitpid(pid,&status,0) == -1)
 {
     printf("wait chile process failed!");
     exit(0);
 }

 ```
 
6. find_command是在go_command中嵌套的操作,是进行是否为有效命令的判断

#### 四,项目演示

1.实现重定向

![](http://img.blog.csdn.net/20170731041659599?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](http://img.blog.csdn.net/20170731041731232?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2.实现管道

![](http://img.blog.csdn.net/20170731041908583?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3.实现shell内置命令(cd为例)

![](http://img.blog.csdn.net/20170731041827969?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

4.阻断Ctrl-C的信号

![](http://img.blog.csdn.net/20170731041938738?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

5.实现后台应用程序

![](http://img.blog.csdn.net/20170731042003705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

6.设置了环境变量

![](http://img.blog.csdn.net/20170731042049248?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 五,存在问题

在此次的项目设计中,也存在许多的问题

如:

1.cd,不能实现直接回家目录的功能,如果使用readline库可能实现会方便一点

2.不支持多管道,多重定向,需要继续进行设计

3.实现的内置命令不够

4.没有成功使用readline库,Tab补全不行,不能上下翻

以上只是其中比较明显的问题,自己设计的shell离bash还有很长的一段距离

在之后得更新中,会逐步解决这些问题

#### 六,设计感想

这次的进程相关设计,使我认识到自己学习的不足

进行编程的学习,书本的知识远远是不够的,

很多时候,自己上手实践的结果要更可靠得多

比如wait函数的调用,在进行了程序设计之后,才对其有明了的认识

在管道功能的实现处,真正了解到进程在程序设计中的重要性

以后的路还很长,还需要继续进行努力

my_shell暂时告一段落,以后有时间会继续进行更新的
