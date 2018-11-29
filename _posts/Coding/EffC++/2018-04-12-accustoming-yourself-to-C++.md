---
layout: post
title: <星月夜> Accustoming Yourself to C++
date: September 7, 2018 5:38 PM
excerpt: Chapter I 让自己习惯C++
categories:
- C/C++
tags:
- EffectiveC++
toc: true
comments: true
---

*C++的世界的确是缤纷多彩的, 这是一门十分强大的编程语言, 随之而来的代价自然就是其使用麻烦*
*本章是全书第一章, 意在使自己熟悉使用C++,所必需要遵守的基础规则*

## Item 01: View C++ as a federation of languages

作为第一条款,首先我们必须承认的一点是:
**C++是一门复杂的多范式编程语言, 它不仅仅是一门语言**

承认这一点, 使得我们明确: C++ 是一个语言联邦, 不能简单地依靠某种语言规范去学习认识它
**而应该根据不同的范式情况, 去进行语言规范的转换, 从而编写高效的C++代码**

目前已经被承认的C++范式有以下几种:

- 面向过程编程
- 基于对象编程
- 面向对象编程
- 泛型编程(Template)
- 函数式编程

那么,我们根据此,可以将C++划分为以下几种子语言:

1. C, (C++的基础)
2. Object-oriented C++, (支持面向对象范式)
3. Template C++, (模板编程, 扩充为泛型编程)
4. STL (Standard Template Library)

根据上面的约定, 我们在使用不同的子部分时, 则需要遵守不同的约定.(理所当然, 对吧 ? ).
举个简单的例子:(EffC++中精巧的例子)

> Q: 函数调用是一门编程语言不可缺少的部分,那么以何种方式进行参数传递?

> A: 我们根据不同的联邦进行划分: 
>
 1. C部分中,我们更倾向于 *pass by value*
 2. Object-oriented 部分中, 我们倾向于 *pass by reference to const*
 3. Template C++ 部分中, 我们一定要用 *pass by reference to const* 
 4. STL 部分中, (因为下面基于指针实现), 倾向于 *pass by value*

上面只是一个小的例子, 但是却反映出了"**将C++视为一个语言联邦思想**"的重要性

> 请记住：
> - C++高效编程守则视状况而变化， 取决于你使用C++哪一部分

## Item 02: Prefer consts, enums, inlines to #define

C++在1990s时, 的确只是作为C语言的扩展版, 增加了基于对象的内容,
也就是写不好C++,经常被人调侃的 *C with class*
但是, 随着语言的发展与完善, C++ 已经成长为一门多范式编程语言, 而继承自C part的预处理器(CCP)
便显得有些"过时"了, (PS: 指有更好地实现方法)
作为现代C++, 我们要尽量减少对预处理器的依赖性, 因为其只是一个简单地文本替换器, 并非语言实现.

我们对于预处理器的依赖经常表现在下面几个方面:

1. \#define PI 3.1415 即常量整型变量的定义
2. \#define max(a, b) (a) > (b) ? a : b 即类函数宏, (PS: 这个max实现的有缺陷)

<1> 实际上是我们为了编程的方便而使用的方法, 减少了代码变更带来的review代价
<2> 则是为了减少使用函数的开销， 执行任务短小的函数，开销十分浪费

但是，以今天的眼光来看(成书于2005年，都已显得“过时”)，我们应该对已作出如下改变：

### const integeral type

对于整型常量的设定，我们建议使用这样的方法：

```cpp
#define PI 3.1415
		#=>
const int pi = 3.1415;
```

原因在于:
使用#define预处理器，只是进行简单的文本替换。PI并不能进入符号表。
无论是进行调试，还是其他操作都十分不方便，PI始终就没有出现过，作为变量来说不合理
(PS: 虽然说const为常量， 这只是编译器的约束，它仍然是可修改的变量)

即使我们使用`const type var` 替代 `#define VER VAL`，也不能完全解决我们的问题，
常见的还有下面两个要注意的要点：
1. 对于C-Style，极其建议使用`str::stirng`。
 因为`const std::string str;` 等价于 `const char * const ptr;` 
2. class中的变量都是与对象相关的， 如果我们在class中声明了const变量。
 它会因为多个对象的存在，而产生多个实体。最好使用这样的手法：
 ```cpp
 class Temp {
 public:
 	const static int pi = 3.1415;
 ...
 };
 ```
 这样就不会产生多份实体拷贝了

下面要说的就是，关于类中static变量，C++11开始全面支持类内初始化
如果是不支持此特性的编译器，有两个办法：
1. 更换最新的编译器
2. 使用类内声明，类外定义的手法

比如这样：
```cpp
class Temp {
public:
    const static int pi;        // declaration
...
};

cosnt static Temp::pi = 3.1415;  // defination
```

而全面支持C++11的编译器可以这样使用
```cpp
class Temp {
public:
    const static int pi = 3.1415;        // declaration
...
};

// DO NOT assign to the static variable; 
const static Temp::pi;  // defination
```

这样就可以比较合理的解决不能类内初始化了。
但是，我们其实还可以使用**enum hack**的手法来处理

> enum hack

> 所谓“enum hack”指的是，使用赋值的枚举变量来模拟类内初始化
> 更关键的是，**enum hack的行为和预处理器类似**，在某些情况下，需要预处理器的功能时，可以通过这样的手法来实现。

那么，enum hack如何实现呢？ 这样写：

```cpp
class Temp {
public:
    enum {PI = 3.1415};
    ....
};
```

*此时为2005年，还没有enum class ( C++11 ), *

**enum hack**有这样两个优点：
1. 它一定不会浪费内存空间，同预处理器一样，不会进入符号表
2. 实用主义来讲，大量代码用到enum hack。 *说的就是你TMP（Template MetaProgramming）*

### #define => Macros

使用预处理器的另一个重要场合就是: 减少函数开销的类函数宏 (getchar(  )的实现方式)

看着很美好，但是预处理器实现的类函数宏，经常是错误百出，各种各样的烦人。

就那上面那个例子来说吧：
**1. 起码每个变量加上括号，同时整体还要加上括号，每步运算也要加上括号**
**2. 更恶心的是，预处理器是文本替换，++， -- 有多可怕用过的人都知道**

基于上面的原因，预处理器实现类函数宏，如今也是一种不良的手法。

使用现代C++，我们应该这样做：`inline template function`

1. 使用`inline`是为了获得与类函数宏一样的高效，同时更获得了类型安全检查
2. 使用`template`在于，我们并不知道参数类型，使用`const T &`理所当然

那么，就来看一个例子吧：
```cpp
#define max(a, b) (a) > (b) ? a : b;

#=>

template <typename T>
inline const T &max(const T &a, const T &b) {
    return a > b ? a : b;
}
```

使用`inline template function`的手法，我们既获得了高效性，还得到了类型安全。
模板泛型的适用场景不一定够多，但inline的使用场景够够的了。

上面说了那么多，我们的目的**并非是禁止使用预处理器**，而是**减少预处理器带来不可预料行为**
毕竟我们当前使用`#include <file>`导入，`#ifndef/#define/#endif`控制编译仍然重要。

> 请记住：
> - 对于单纯变量，最好以const对象或enums替换#define
> - 对于形似函数的宏（macros），最好改用inline函数替换#define

## Item 03: Use const whenever possible

在item 02 中，我们已经见识到const所带来的便利了。

现在就再来谈谈const，后面还会聊到它的。

关于const，我首先想要说说这两点：
1. 所谓const，只是编译器/程序员的“约束假定”，保证从编译器/程序员的方面不去修改
 但是，实际上是可修改的。这点在后续关于const的讨论中很关键
2. const在星号左是底层const，修饰指向。在星号右是顶层const，修饰变量本身（引用。。）

有了上面的基础后，我们再来聊聊const。

关于一般的内置类型使用const，我们就不再赘述了

重点来说说类中的const => const函数

### const function

提到const函数，有这几个：函数参数，返回值，函数被const修饰。
**const函数，狭义上特指const的成员函数，表示我们并不会修改此对象，**

在这里我们要考虑的重要内容是：**const属性可以进行重载**

我们在设计类的接口时，应该合理考虑到会遇到的参数类型，从而设计const及non-const版本
比如我们会遇到这样的场合：
```cpp
class Library {
public:
    char &operastor[](char *str);
    const char &operator[](const char *str);
};

void func(const char *str)
{
    std::cout << str[2];
}
void func("hello ,world");
```

这其中就使用了const版本的接口。
若使用const版本时，返回值并非`pass by reference to const` 而是 `pass by reference`
就会出现，const对象被意外修改的情况。
比如：

```cpp
const textString t("hello");
char *ch = &t[0];
*ch = 'J';   #=> const object t has been modified
```

滥用cosnt，并不会对导致出现成员函数异常，但是会出现这样的错误。

这就涉及到*bitwise constness* 与 *logical constness*

上面出现的问题本质是：**在bitwise constness中改变了指针的指向，未改变指针**
**已经违背了logical constness，但是在编译器的bitwise constness层面上非错误**

**上面是个隐式的错误接口开放，const内部，结果导致了非const接口开放，破坏了类的常量性质**

对于， 这种问题的解法便是：**使用mutable来将成员常量性分离**
即使在const对象的内部，mutable类型的变量仍然是可以被修改的！

### const and non-const

在提供多个版本接口时，不可避免的是，代码的相似性与重复劳动(对程序员)
我们的建议就是：使用已经实现的部分来管控未实现的部分。
针对const与non-const要求是：**使用const来实现non-const**

原因便是：**const版本做出承诺不修改对象，non-const可以在调用const时，正常工作**
**反之不可，因为non-const不承诺，我们用non-const来减少const的重复工作没有承诺，原来保证不变的对象可能发生改变。**

**而且： 因为我们在此处已经确定转型是安全的，所以使用`static_cast<>`**

```cpp
const char &operator[](std::string::size_type pos)
{
	return str[pos];
}

char &operator[](std::string::size_type pos)
{
	return const_cast<char &>
    				(static_cast<const Temp &>(*this)[pos]);
}
```

另外，这种消除重复的手法，还可以用于：使用operator==() 实现 operator!=()之类

总之，const是个非常灵活而又实用的东西，多多使用，多多发掘。

> 请记住：
> - 将某些东西声明为const可帮助编译器侦测出错误用法，const可被施加于任何作用域内的对象，函数参数，函数返回类型，成员函数本身
> - 编译器强制实施bitwise constness，但编写程序应该使用“概念上的常量性”，即注意logical constness的问题，以及mutable的使用
> - 当const和non-const成员函数有着实质上的等价实现时，令non-const版本调用const版本实现可以避免代码重复

## Item 04：Make sure that objects are initialized before thre're used

*这一部分其实是一个常识，没错，就是使用前初始化！*

简单的来说，按照下面几个步骤走下来，一定可以保证正确性和合理性：

1. 内置类型，手动初始化
2. 自定义类类型，依靠构造函数初始化，其中一定要注意初始化列表的问题
3. 处理好初始化次序不定的情况(不同编译单元中的non-local static 变量)

### 内置类型

其中内置类型的初始化，你我也知道 ---- 默认初始化(与值初始化区分)。都是未定义的值(..)

### 自定义类类型

我们强调一下构造函数，来看看下面两种形式的构造函数：
```cpp
class Text {
public:
	Text() = default;
    Text(std::string text, std::string author, int num, double price);
private:
	std::string textName;
    std::string authorName;
    int 	    textNum;
    double 	 textPrice;

}

// ver.1
Text(std::string text, std::string author, int num, double price)
{
    textName = text;
    authorName = author;
    textNum = num;
    textPrice = price;
}

// ver.2
Text(std::string text, std::string author, int num, double price)
	: textName(text), authorName(author), textNum(num), textPrice(price)
{ ; }
```

ver.1 看上去更像我们传统意义上的初始化，**其实是假的，这个叫赋值**

ver.2 则是真正意义上的初始化。

有这样几个点是要了解的：
1. C++是为数不多严格区分赋值与初始化概念的语言的(可能是我见得少)
2. C++中初始化，发生在赋值之前，或者说在构造函数值之前
3. **我们墙裂建议使用member initialization list的形式初始化成员**

主要出于下面的考量：
1. 初始化(拷贝构造)和default构造 + 赋值的开销一目了然，肯定是拷贝构造低
2. 即使自定义类型的拷贝构造和 defauilt + 赋值开销近似，但为了一致性
3. 如果有的类，真的丧心病狂，没有default构造了，玩锤子ver.1

接下来，我们要注意的是初始化的顺序：
**1. 函数调用上，初始化在构造之前，base-class在deriver-class之前**
**2. 具体成员与类内声明顺序相关，与list顺序无关，所以，比要互相依赖初始化**

除了某些特殊情况：比如从文件，数据库读入东西时，使用ver.1形式。
支持，第二个过程，自定义类类型的初始化也确定了

### 不同编译单元的non-local static varible

最恶心人的便是：不同编译单元中的non-local static变量的初始化顺序，恶心的要死
**可以说是，根本无法确定这样的顺序，因为不同的人，在不同时刻编写的代码**
**根本无法确定初始化次序，这是个无解的难题**

有句话说得好，，*办法总比困难多*
借用《Design pattern》 中单例模式的讨论，我们可以使用技巧规避这个问题

先来补点基础：
**function内部的static变量叫做local static var，因为他一定经历了初始化，有定义式**
**其他的叫做non-local static var，如namespace，global，file中的，都是从定义出产生，到程序结束消亡**

**关键就是，你使用这样的变量时，他未定义，凉了。。。**
我们的思路便是: 将使用特定的non-local static var转换为local static var使用

也就是说，采用这样的手法：
```cpp
//file1.cc
class FileSyatem {
public:
	...
private:
	...
};

Filesystem &get_filesystem()
{
	static Filesystem fs;
    return fs;
}

// file2.cc
class Directory {
public:
...
private:
...
};

Directory &get_dir()
{
	static Directory dir;
    return dir;
}

// file3.cc
// Now the filesystem, directory object must be initialized
Filesystem global_fs = get_filesystem();
Directory global_dir = get_dir();
```

PS: 一个小问题，关于竞争状态（Race Condition), 只要是non-const的static
**无论是local，或non-local，都会引发竞争状态**
解法：**单线程阶段，完全初始化完成即可避免**

总之，只要完整按照上面的三步走，就可以基本保证适用对象前初始化的问题！

> 请记住：
> - 为内置类型对象执行手工初始化，因为C++不保证初始化他们
> - 构造函数最好使用成员初始化列表，初始化列表列出的成员变量，其排列次序应该和他们在class中的声明明次序相同
> - 为免除"跨编译单元之初始化次序"问题， 请以local static对象替换non-local static对象
