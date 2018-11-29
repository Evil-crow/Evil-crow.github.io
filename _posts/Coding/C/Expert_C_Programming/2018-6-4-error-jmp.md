--
layout: post
title: 聊一聊C中的异常处理
date: June 4, 2018 12:02 AM
excerpt: 今天在了解C++中的异常机制的时候,想起来C中实际上也是存在异常处理的,只不过我们平时用的比较少而已,而C中的异常处理机制,主要依赖于setjmp()以及longjmp()这两个函数,下面我们来看看这两个函数的用法,以及Linux内核中的异常处理机制
categories:
- C/C++
tags:
- Expert_C_Programming
toc: true
comments: true
---

*所谓异常处理机制,其实就是处理程序可能会发生地错误.*

*那么,我们的主要步骤也就是两个: 抓取异常, 处理异常*

*C++中我们使用throw(抛出异常)以及catch(捕获及处理异常)*

*C中我们使用longjmp()来进行抛出, setjmp()来处理异常*

## 一, setjmp()与longjmp()

```cpp
int setjmp(jmp_buf env);
int sigsetjmp(sigjmp_buf env, int savesigs);

void longjmp(jmp_buf env, int val);
void siglongjmp(sigjmp_buf env, int val);
```

这两个函数使用来实现非本地跳转的,即"non local gotos"

*将执行从一个函数转移到一个函数在另一个函数中的预定位置,setjmp()函数动态的建立将要控制的目标*

*而longjmp()函数执行转移*

**我们说setjmp()函数用来建立将要控制的目标,那他怎么建立呢?**

通过查看man文档以及 /usr/include/setjmp.h可知

```cpp
/usr/include/setjmp.h

#include <bits/setjmp.h>        /* Get '__jmp_buf'.  */
#include <bits/types/__sigset_t.h>
 
/* Calling environment, plus possibly a saved signal mask.  */
struct __jmp_buf_tag
{
    /* NOTE: The machine-dependent definitions of `__sigsetjmp'
       assume that a `jmp_buf' begins with a `__jmp_buf' and that
       `__mask_was_saved' follows it.  Do not move these members
       or add others before it.  */
    __jmp_buf __jmpbuf;     /* Calling environment.  */
    int __mask_was_saved;   /* Saved the signal mask?  */
    __sigset_t __saved_mask;    /* Saved signal mask.  */
};

// 上面的结构体成员,分别是调用环境(保存堆栈指针, 指令等), 是否保存标志位的标志, 保存的信号掩码
// 而且,注释严重声明: 前两个成员是不能互相交换位置的.
// 如果要详细查看__jmp_buf 可以去看/usr/include/bits/setjmp.h
```

即: setjmp()的参数,jmp_buf实际上就是保存现场的,而longjmp()就是用来恢复现场的.

**保存现场,就是保存活动记录...**

其中sigsetjmp(), siglongjmp(), 即可以理解为signal-setjmp(), signal-longjmp()

*只是额外提供了对进程信号的可预测处理, 在此不表,用法与non-signal版本差不多,不多说*

**介绍了这么多,那么这两个函数如何使用呢?**

我们可以从他们的返回值入手

**setjmp()直接返回值,也就是第一次调用为0, 之后longjmp()无返回值,但是longjmp()的参数val**

**作为setjmp()第二次之后的返回值, 若未指定val,setjmp()之后返回的不是0, 而是1**

**但是,setjmp() + longjmp()的严重问题是,他也不能随意跳转,只能是setjmp()保存过活动记录的地方**

*其实有过C++经验的,可以勉强将setjmp(), 看成try字句*

## 二, Notes:

1> 注意

其中需要注意的是,jmp系列函数是通过保存现场来实现非本地跳转的,其中需要注意的一点就是:

**编译器会将变量优化为寄存器变量,所以在调用longjmp()之后,自动变量的值是未指定的**

**为什么这样讲, 因为你使用longjmp()的时候,jmp_buf的内容被销毁**

**所以自动变量(存在register中)的就凉凉了**

如果满足下面的条件:

1. 相应的setjmp()函数是局部的

2. 自动变量值在setjmp()以及longjmp()之间发生了变化

3. 变量没有被声明为volatile (反之,也就是说变量声明为volatile就不会被优化至寄存器)

**则我们要求使用siglongjmp()函数**

2> 可读性分析

关于可读性是一个大问题, 尽管non-local gotos可以被随意使用, 但是相比于local gotos,

**goto字句,起码还有goto语句以及label的提示啊**

但是,jmp函数,甚至多个longjmp()共享同一个jmp_buf变量进行操作

另外有一点: 甚至setjmp()以及longjmp()不在同一个文件模块中

**所以,我们建议除非迫不得已,请使用其他的替代方案**

3> 警告

1. 如调用一个函数,此函数调用了setjmp(),却在longjmp()之前返回,那么它的行为是未定义的

2. 在多线程程序中, 如果一个longjmp()函数使用了不同线程中的jmp_buf,那么它的行为同样是未定义的

此外,讨论的一些,非异步信号安全函数,此处暂时不涉及,不清楚,之后有机会会填坑

## 三, non-local gotos的例子

```cpp
//main.c
#include <stdio.h>
#include <stdlib.h>
#include <setjmp.h>
#include "set.h"
int main(void)
{
    int ret_val;

    if (ret_val = setjmp(jmp)) {
        printf("Back to main.c and ret_val is %d\n", ret_val);
    } else {
        printf("First time through\n");
        printf("The ret_val: %d\n", ret_val);
        test();
    }

    return 0;
}
```

```cpp
// set.h
#ifndef _SET_H
#define _SET_H          
#include <setjmp.h> 
jmp_buf jmp;     
void test(void);
#endif

// set.c
#include <stdio.h>
#include <stdlib.h>
#include <setjmp.h>            
#include "set.h"               
     
void test(void)                
{    
    printf("This is set.c\n"); 
    longjmp(jmp, 4);           
    printf("YOu can't get here!\n");
}
```

结果如下,大家看了代码基本就懂了:

![./a.out](http://p8uroi1uf.bkt.clouddn.com/jmp.png)

## 四, Linux内核的错误处理

大家可以去看看代码,实际上内核的错误处理是使用多重goto来实现的

比较下面两种形式:

```cpp
if (error1) {
	do_something();
} else {
	do_something();
}
if (error2) {
	do_something();
} else {
	do_something();
}

```

```cpp
do_something();
if (error1)
	goto error;
if (error2)
	goto error;
	
error:
	do_something();
```

**很明显的可以看出,第二种的做法着实方便,提高了程序设计的维度,不会影响设计的重心**

**将错误处理集中在其他地方**

June 4, 2018 8:57 PM
