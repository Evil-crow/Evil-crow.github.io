---
layout: post
title: 语法陷阱与词法陷阱
date: January 21, 2018 8:12 AM
excerpt: C陷阱与缺陷是一本深入剖析C语言常见问题地书籍,值得不断的回味
categories:
- C/C++
tags:
- CTrapsAndPitfalls
comments: true
---

*最近重装了系统,/挂载在固态里面是真的快,但是,KDE的中文字体现在搞得我想吃shi,楷体是真的难受*

(upate:适应了几天,还不错,谁的Linux有我的楷体sao(滑稽));)

*不扯了,不扯了,C陷阱与缺陷是很好的一本书,这一篇是词法与语法中的陷阱*

*其中有几个是很多人整天苦恼地问题呢*

# 词法陷阱

## 1. = 不同于 ==

这个问题,基本上只要是使用过C语言的都是知道的,赋值运算符与比较运算符是不能混淆的

尤其是在,选择或者循环控制语句中,混淆使用会导致不可预料的后果

但是,平时应该如何规避这样的错误呢?

显式的使用括号进行判别

**一般情况下,在if while等控制语句中需要的是 == 运算符,但是遇到必要的使用 = 进行判断的时候呢?**

```cpp
/*检查新值是否为0*/

if (a = b)
	foo( );

可以写作:

if ((a = b) != 0)    //显式的表示更加明确
	foo( );
```

## 2. & 和 | 不同于 && 和 ||

在之前看C的书籍与CS:APP都提到了这个问题

只有在限定数值范围的情况下,才能得到相同的结果

## 3. 词法分析中的"贪心法"

其实只要了解过,正则表达式的人,就可以很方便的理解贪心法

即是指:**进行词法分析时,尽可能多的进行运算符的匹配**

比如:

```cpp
a---b #=> a -- - b

y = x/*p #=> y = x /* p                            */

二义性,应该这样写:

y = x/(*p)
```

而且,在ASCI标准没有出现之前,会有许许多多尴尬的情况

**准确得讲,因为其中不标准的规范,才会有上古时期很难阅读的代码**

**而在现在,基本上都支持ASCI标准,所以,好用多了**

## 4. 整型常量

众所周知,在C中支持,二进制,八进制,十进制,十六进制,**采用不同的书写方式**

所以,会出现下面这样的情况,**将十进制误写成其他进制**

```cpp
struct {
	int part_number;
	char *description;
}parttab[] = {
	046, "left-handed weight",
	047, "right-handed weight",
	125, "frammis"
};
```

上面便出现了将十进制误写成八进制的情况,需要严加防范

## 5. 字符与字符串

...

# 语法陷阱

## 1. 理解函数声明

函数声明,是C中进行程序设计所必不可少的,

有难度吗?

**有,当你遇到,函数指针,返回值为指针的函数,嵌套时,有你好受的 ! **

来看一个例子,这是选自 < C陷阱与缺陷 > 的例子

```cpp
(*(void(*)( ))0)( );
```
上面这个东西是真的看的人头疼!

那么,我们拆开来看吧!

首先,是这一部分,

```cpp
(*(...)0)( );
```

可以清楚的看到,

**如果没有后面的0,那么便是进行函数指针的调用,比如 (*fp)( );**

**但是,加上0,就不能解释为简单地函数指针调用了,而是进行强制类型转换!**

**即,此处是将0,转换成为一个函数指针,这样才能实现需要:调用0位置的子例程**

再来看,括号内的内容

```cpp
(void(*)( ))
```

这里就要介绍一种技巧了,**即是获取到类型转换符的方法**

很简单:**将标识符,分号,去掉之后.用括号封装剩下的内容即可**

来举个例子:

```cpp
void (*fp)(ARGV);      /*其中括号是必要的,因为优先级*/
```

上面的例子,便是一个返回类型为void的函数指针,函数指针标识符为fp,

那么,0便是要接受这样的类型转换,将上面的式子转换为类型转换运算符即为

```cpp
(void(*)())
```
这样便可以作为0的类型转换符来使用了

表示含义为"返回值为void类型的函数指针的强制类型转换"

现在回过头来,我们再整体分析一遍

```cpp
(*(void(*)( ))0)( );

(*(强制类型转换为指针,返回值为void)调用地址0处)(函数参数);
```

上面因为直接进行对0地址处的进程调用,所以写出来的是,强制类型转换为指针类型,

同时也是因为,此处要操作的是函数指针的缘故.

这样的形式是真的复杂,所以我们可以使用typedef来进行别名命名

```cpp
typedef void(*fp)( );

(*(fp)0)( );
```

同时,**也让我更深刻的理解了,typedef的内容,不仅仅命名结构体,还可以命名函数指针**

上面是一个简单地例子,下面我们来看一个更, , ,的例子,库函数中,signal函数的实现

**首先来说说需求: 传参为int型的信号值,另一个参数,值用户制定的函数指针**

首先,我们来设计一个基础的版本

```cpp
void signalfunc(int );
```

很简单对吧,就是一个返回值为空,传参为int值即可

我们现在需要完成一个需求,声明一个指向signalfunc的函数指针

```cpp
void (*sfp)(int);
```

没错对吧,sfp即为指向signalfunc的函数指针

所以,我们的signal函数则可以这样写:

```cpp
void (*signal(ARGV))(int);
```

其中ARGV是signal的参数,那么signal函数的参数是什么呢?

一个int型的信号值,一个用户自定义的函数

那么,这就简单了,ARGV可以展开写成

```cpp
void (*signal(int ,void (*)(int)))(int);
```

所以,现在来看,我们可以这样理解:

**使用函数指针,是为了进行简单的函数调用设计,的那是因为C语言特性**

**一直以来使得,C函数原型(过去叫函数声明)理解晦涩,实质上,可以使用递归德斯向来理解(Pangda)**

**即,将多层函数指针嵌套的函数原型,进行按层拆封,由内向外,一层一层的进行划分**

**其中比较麻烦的就是,函数的参都不一致,所以看起来难以辨别,这种时候使用typedef就清晰明了了**

比如:

```cpp

typedef void (*HANDLER)(int);

void (*signal(int, void (*)(int)))(int);

#=> HANDLER signal(int, HANDLER);
```

```cpp
void *(*k(int p, int(*l)(int, void(*m)(int))))(int o);
```
上面这样的形式,一般是比较迷惑人的,

仔细划分,函数名就是k,就是第一层,**其他全部是调用的函数指针的形式参数名**

**这一部分,十分重要,可以让你以后设计出,厉(ri)害(gou)的函数**

## 2. 运算符的优先级

|运算符|结合型|
|-----|-----|
|( ) [ ] -> .|自左向右|
|! ~ ++ -- - (type) * & sizeof| 自右至左|
|* / %|自左向右|
|+ -|自左向右|
|<< >>| 自左向右|
|< <= => >| 自左向右|
|== != | 自左向右|
|&|自左向右|
|^|自左向右|
| ｜|自左向右|
|&&|自左向右|
|｜｜|自左向右|
|?:|自右向左|
|assignments|自右向左|
|,|自左向右|

上面的便是,运算符优先级表,

**反正我觉得不会好记**,一般有两种办法处理优先级问题

**1. 加括号, 2. 背表**

## 3. 注意语句结束标志的分号

C中使用" ; "作为语句的结尾,让人又爱又恨

好的地方在于:**清晰的标志了语句地结束**

不好的地方在于: **用错的时候,会出现爆炸般的错误**

来看下面这几个例子:

```cpp
if (a > b);
	func( );

等价:

if (a > b) { }
	func( );
```

相信很多人都出过这样的错,提前结束判断/循环语句,导致程序出现不可控的错误

**当然,有时候,有必要的话应该这样写**

```cpp
if (a > b)
	;
func( );
```
写空语句的形式,更加清晰明了,

**想要以格式来进行程序的逻辑控制,在C中是痴心妄想,游标卡尺倒是还可以**

**多分号,很危险.同理,少了分号也不得了,来看下面的例子**

```cpp
if (a > b)
	return 
a.one = 1;
a.two = 2;
a.three = 3;

等价:

if (a > b)
	return a.one = 1;
a.two = 2;
a.three = 3;
```

```cpp
struct test {
	int a;
	float b;
	struct test *next;
}

main(void)
{
	...
}

则指定了main函数的返回值为test结构体,在一定的情况下,会出现爆炸现象的,(此处不会使用缺省类型int)
```

## 4. switch语句

关于switch语句,需要注意的点在于:

**关于逻辑控制的break语句**

**切记,没有break,则会顺序执行**

~~小组面试也考过的~~

当然没有绝对的错

比如在写一个运算器的时候:

**减法也可以转换成为加法**

```cpp
switch (operation) {
	case '-':operation = -operation;
	//此处没有break;
	case '+':....
	....
}

这种时候加上注释,效果更好呦
```

## 5. 函数调用与else

~~其实就是意思一下,大家都懂得~~

**C中函数调用,必须有括号,不然只是计算函数的地址,并不调用,一般linter都会报错**

**else,与最紧密相联的if构成一对(在括号不清晰时),还是那句话,不要妄图在C中间加入py那种玩艺(滑稽
**

这本书,怎么说,还是因为放假回来,一开始看不进入其他的书

不过,怎么说,这本书,**还挺好看的,~~(虽然买的是盗版,不过也是钱啊0)~~**

下一章开始就好玩了

**~~另外,我也想像瑞神一样,分析一波C标准库,起码看完,C自身的特性就都懂了~~**

**像Pangda那种OOP的C,我还是不想玩了,dan teng,不如直接rb**

January 23, 2018 11:41 AM