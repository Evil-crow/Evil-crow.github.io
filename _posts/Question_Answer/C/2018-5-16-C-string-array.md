---
layout: post
comments: true
title: C/C++ char *与char [ ]
date: May 16, 2018 10:47 PM
excerpt: 唉,又被教育了,C-String, C\+\+ (cstring/String),差得还是不少,C\+\+学习越来越长
categories:
- Q&A
---
*学C\+\+并没有那么舒服, 怪不得是太阳系最复杂的语言,脑子疼*

来说说今天被教育的吧.

先从这样一条语句开始:

```cpp
char *str = "XiyouLinuxGroup";
```

看上去稀松平常, 对没错, 实际上就是稀松平常

在C语言中有这样要注意的:

**1. str真的只是指向一个字符的指针, 只是X字符的指针, sizeof结果也只是8(字长)**

**2. char str[ ]的形式就是正常的数组, 包含了整个字符串, sizeof的结果就是字符串长度 + 1**

那么,到了C\+\+中就是这样的:

**1. 类型推导**

```cpp
str <= char *
"XiyouLinuxGroup" <= const char *
```

这样一来,只要编译就会有警告

```bash
ISO C++ forbids converting a string constant to ‘char*’ [-Wwrite-strings]
```

**甚至在C++11之后编译不能通过**

那么, 有什么办法呢?

```cpp
1. char str[] = "XiyouLinuxGroup";
2. const char *ptr = "...";
```

**而且, 一般在使用时,不想进行修改, 就加上const了**

**最后, 一般在对于字符串的操作, C\+\+中可以尽量避免C-String出现,**

**但是因为开销小, 适当情况下, 还是要使用的**

再来说说delete

```cpp
delete str;    // 释放一个字节的空间
delete []str;  // 释放str一系列的空间
```

**最后, 关于释放空间的事, 栈上的不用手动处理, 分清何时需要分配, 何时需要释放还是很重要的**

**C\+\+学起来, 真的不轻松, 还是继续努力吧 ! **

May 16, 2018 11:02 PM
