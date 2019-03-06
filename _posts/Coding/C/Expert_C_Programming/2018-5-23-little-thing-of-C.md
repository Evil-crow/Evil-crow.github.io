---
layout: post
title: <咸鱼书> C中的一些小细节
date: May 23, 2018 12:00 AM
excerpt: < C专家编程 >第二章其实没什么意思,所以,我就改了一下标题,名为,C中的小细节
categories:
- C/C++
tags:
- Expert_C_Programming
toc: true
comments: true
---

*正如概述所言,第二章没啥意思,所以我决定改名,挑出其中核心的部分,没错就是这样,就有了C中的小细节*

为了方便表述,下面的内容我用点来表示:

1. fall-through	的特性,这个是switch-case语句设计的问题,注意切换case时,很少有"没有break"

 关于break,我们常见的一个错误就是,break到底跳出了什么?

 答: 实际上跳出的是,最近的一层循环或者switch-case语句

2. core dump的错误,实际上就是段错误,也就是在支持虚拟内存的机器上,如果访问到非法内存,缺页中

 断后,不能正确的页面置换,就发生了这样的错误,而不支持虚拟内存的机器,就没法这样,如MS-Dos

 不过,MS-Dos有它的一套进行非法内存访问的判断措施
 
 ![core dump](http://www.qiniu.evilcrow.site/Exceprt_C_core_dump.png)

3. 缺省可见性,语言实现起来十分复杂,况且C/C++还支持分离式编译,所以文件内容的可见性十分重要

 而在C中,只能这样: 所有缺省的可见性是extern, 而且只有extern,static两种可见性要么可见,
 
 要么不可见.所以在C++中才提供了其他的一些可见性限定符.
 
4. 一般情况下,,彻底使用fgets()替代gets()

5. C中表达式求值是未定义行为,求值顺序不定. 是为了可以让编译器选择最适合的求值顺序,提高效率

6. 关于返回局部变量的问题

来看一段代码:

```cpp
#include <stdio.h>
#include <stdlib.h>
char *local_time(char *filename)
{
    struct tm *tm_ptr;
    struct stat stat_clock;
    char buffer[20];

    stat(filename, &stat_clock);

    tm_ptr = localtime(&stat_clock.st_time);

    strftime(buffer, sizeof(buffer));

    return buffer
}

```

上面代码有什么问题吗?

**对,没错.返回局部变量会造成大问题,已经释放的变量,返回野指针,进行操作和访问就会出错**

我们提供下面这些可考虑的解决方式:

1. 返回指向字符串的指针.

	**强调一下,一定是指向字符串常量,一般的字符串也没啥大变化**
	
2. 使用全局变量数组 (不多说了)

3. 使用静态数组

	函数结束也不会轻易释放,可以返回此指针
	
4. 显式分配内存,保存返回值,即分配得到的内存首地址

5. 手动控控制分配和释放, malloc, free建议在一个块中操作,是最简单的内存管理策略


小细节就这么多,下一章将讨论一下,如何看C的声明 (滑稽
