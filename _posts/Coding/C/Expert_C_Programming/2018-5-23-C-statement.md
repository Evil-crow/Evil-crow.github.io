---
layout: post
title: <咸鱼书> C声明问题
date: May 23, 2018 11:16 PM
excerpt: C的声明向来晦涩,今天就来撕它一撕
categories:
- C/C++
tags:
- Expert_C_Programming
toc: true
comments: true
---

*既然是要撕C语言的声明,那么我们就直接一点吧*

但是,开始之前先说说其他的:闲聊

### 如果有人说,函数参数从右向左压入堆栈,那你就去把他头打烂

 参数传递进入函数时,不是一定都在堆栈区的,int型这类数据,是可以进入resigter的,

 而形如结构这样的,是进入堆栈区的,所以说参数传递首先进入寄存器,不是无脑进入堆栈区的

### 聊一聊联合的那点事(union)

 联合一般是用在结构体内部,用来进行节省空间的,因为Union中的数据是不可能同时出现的

 也就是说,union中的数据都是互斥的.联合可以将一个东西解释为两种含义,比如:

```cpp
union test {
    char a;
	int b;
};
void test_edian(void)
{
    union test temp;
	temp.b = 0x12345678;
	printf("%d", temp.a); // 12 Big-endian, 78 little-endian
	
	return 0;
}
```

### 关于枚举(enum)

枚举,其实平时用的是比较少的,但是enum还是有它的好处,#define经常与他混用

Linux Kernel 2.6X时,伙伴算法中的flags标志位使用#define, 整整15个

Linux Kernel 3.x后,全部修改为枚举变量,无中生有吗?

并不是,是因为枚举相对于#define有这样一个巨大的好处:

\#define在预编译阶段结束之后,便丢弃.而枚举依然能够保存在符号表中,之后方便调试

也即是说,#define就在符号表中找不到了,这样就简单了

C声明的那点事

下面进入我们的正题,如何来看C语言的声明优先级

首先看看这张图:

![如何解析C语言声明](http://p8pmsq2a4.bkt.clouddn.com/C_statement.png) 

神图!,我来给大家解释一下.

1. 从标识符看起

2. 首先向右看,如果是方括号,则为数组.如果是括号,就是函数,再去看参数(方括与括号不可能同级出现)

3. 之后向左看,如果是一个左括号,则此部分声明已经被组合到一起,去找此括号,对应的右括号

4. 之后回到第2步,继续

5. 再次到第3步,如果是const, volatile *之一,就重复继续看是否有左括号

6. 直到最后,遇到基本类型可以一次读完.

光说不练假把式,来分析三个实例:

```cpp
char * const *(*next)();

char *(* c[10])(int **p);

void (*signal(int sig, void(*func)(int)))(int);

```
第一个实例:
-> 标识符为next -&gt; 右看没有 -&gt; 左看是”*” -&gt; 得到: next时一个指针,指向XXX

-> 左看是左括号,找到匹配右括号,之后向右看,括号,即为函数,所以next是一个函数指针

-> 左看,*, 指向…的指针 -&gt; 继续, 只读属性 -&gt; 继续, 指向…的指针 -&gt; 继续, char类型,结束

\#=> 得到结果: next是一个函数指针, 他指向一个参数为void, 返回值是指向只读字符串的指针

第二个实例:

-> 标识符为c

-> 右看,[],c是一个数组

-> 右括号,向左看, *,则数组元素为指针,c[10]里面存了, 10个 指向XXX的指针

-> 出来,向右看,括号,函数,参数就那样,结束

-> 向左看, char * 为返回值

\#=> 得到结果: c是一个数组,里面放了10个指针,10个指针指向了函数,char (\*)(int \*\*)类型 c即为函数指针数组

第三个实例:

-> 标识符为signal,右看( ),即为函数

-> 函数参数为(…int及函数指针)

-> 右括号, 则向左看, *,为signal函数的返回值,是一个指针

-> 指针指向void (*)(int)

这个就比较复杂,但是我们可以这样:

```cpp
 typedef void (*pfn)(int);
 
 signal #=> pfn signal(int, pfn);

```
 从这里我们就可以看出,我们使用typedef可以将函数指针类型左为返回值剥离出来
 
 C这种类似于递归理解的声明语法确是看的人眼花缭乱,一切还是因为他晦涩的语法的原因
 
###  typedef的前身后事

既然这样,我们就再来说说typedef: typedef原名存储类型说明符

他常用的地方在于: 对已存在数据类型进行重新命名(别名), 方便大型程序的开发

用法: 像平常的变量声明一般,加上typedef即可.

```cpp
typedef int a;

typedef void (*pd)(int)
```

即可完成,当然了,需要加上分号( ; )

同一个代码块中,typedef不能引入和其他标识符重名的名字

typedef有两个重要的问题:

```cpp
1. typedef int *ptr, (*fun)(), arr[5];  //分别为int指针, int(*)()函数指针, size=5的int数组
2. unsigned const long typedef int volatile *kumquat; // 不要在中间使用</span>
```
至于typedef与#define的区别:

1.mtypedef是彻底的”封装”, 可以使用#define进行类型扩展, typedef无效

```cpp
typedef char * cstring;
define char * CSTRING
const cstring str;   //error
const CSTRING STR;   // correct
```
2.连续几个变量声明,typedef可保证变量类型一致性,#define无法保证

```cpp
define INT_PTR int *
INT_PTR a, b;    #=> a = int *, b = int 
typedef int_ptr int *;
int_ptr c, d;       #=> a = int *, b = int *
```

C声明是个恶心的东西,到了C++也不一定能够完全跳脱, 但是多理解,配上typedef,可以事半功倍

* * *

update: 

另外我要喷一下垃圾js,劳资这篇文档就是因为它的评论系统搞死了,

让我从git commit中把HTML改回来,累死我了,靠!
