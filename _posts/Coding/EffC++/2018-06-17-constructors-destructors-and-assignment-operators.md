---
layout: post
title: <星月夜> Constructors, Destructors, and Assignment Operators
date: September 13, 2018 10:49 PM
excerpt: Chapter II 构造/析构/赋值运算
categories: [C/C++]
tags: [EffectiveC++]
comments: true
---
* TOC
{:toc}

*上一篇中,我们着重进行了C++中最最基础,和C有很大区别的地方,比如语言联邦, 减少预处理器的使用,*
*多使用const, 保证使用前初始化对象等几个方面*

*这一部分, 我们就来聊聊使C++语法变得如此复杂的"罪魁祸首"之一 ---- 复杂的拷贝控制*

## Item 05: Know what functions C++ silently writes and calls

*这是一个关键的点, 我一直因此觉得C++足够傲娇.C/C++有着强大的性能,同时他们又兼具着许多风险行为*
*如何高效编程, 同时规避风险, 我觉得这就是C/C++的美感之一*

我们在此处要讨论的是: **在你不知道的情况下, C++默默编写并调用了哪些函数**
(PS: 此处重要针对讨论的是 `class`内部的情况, global作用域内, 不可能被凭空塞函数的)

先来看一个例子:

```cpp
class Temp { };
```

上面是一个简单地例子: `class Temp {};`我们并没有为它编写任何的成员函数, 那么它可以使用吗?
比如:
`Temp a;     // Constructor`
`Temp a(b);  // Copy-constructor`
`Temp a = b; // Copy-operator-assignment`
`{ .... }    // Destructor`

经过实验, 你可以发现, 上面这些操作都是成功的!
为什么? 我们明明没有编写这些成员函数的啊! 这便是**C++默默编写并调用的函数**

总共有下面这四(六)类:
1. Constructor
2. Copy Constructor
3. Copy assignment operator
4. Move Constructor        (C++11)
5. Move assignment operator (C++11)
6. Destructor

这些默默编写的函数存在着下面的特点:
1.public && inline
2.默认初始化 && 逐次拷贝
3.编译器生成的版本都是non-virtual的, (继承得来的除外)

暂且记下这些特点, 我们下面的讨论会用到, 并非分析如何处理此情况

所以上面的例子实际上是这样子的:

```cpp
class Temp {
    Temp();
    Temp(const Temp &);
    Temp &operator=(const Temp &);
    Temp(Temp &&);
    Temp &operator=(Temp &&);
    ~Temp();
}
```

那么, 编译器一定会生成这些函数吗? 是不是得思考一下?

我们有下面的结论:
**assignment operator不一定会生成, 其余的函数照样会生成**

考虑这样的情况:
1.内含reference的class,
2.内含const成员的class
3.base class的assignment operator为private

来解释一下这三种情况:
1.若含有reference, 拷贝赋值函数到底是修改reference(错误)还是其所指的内容(无意义)?
2.const成员怎么可能会被修改!
3.若base class的拷贝赋值为private, derive class的拷贝赋值函数便无法生成

**简单地来说, 我们设身处地的为编译器着想, 它是否可以处理这种情况, 从而进行判断**
**上面的情况便是: 编译器无法为这些情况处理, 所以它的copy assignment operator创建失败**

> 请记住
>
> 编译器可以暗自为class传播创建default构造函数, copy构造函数, copy assignment 操作,
> 以及析构函数, (以及move 构造函数, move assignment操作 C++11后)

## Item 06: Explicitly disallow the use oof compiler-genreated functions you do not want

*这一条款的讨论是基于 Item 05的, 处理编译器生成函数的不二法宝*

**首先强调一个事实: 如今可以明确的说明对于编译器生成函数我们持有的态度**

*还是hepangda的说法比较准确, " C++11之后语言的语义更加明确 "*
*解释起来就是, C++之前的一些做法, 你不是很明确它的意思, 在一定条件下, 得借助注释才能迅速的理解*

那么,我们先来说说在现代C++中如何处理这种问题吧,

现代C++中我们可以使用**default, 以及delete**来进行编译器生成函数的控制, 表明我们的态度

如下例子:
```cpp
class Temp {
    Temp() = default;
    Temp(const Temp &) = delete;
    Temp &operator=(const Temp &) = delete;
    Temp(Temp &&) = default;
    Temp &operator=(Temp &&) = delete;
    ~Temp();
}
```

**通过语义明确的现代C++, 我们可以做到不想编译器自动生成的函数, 明确拒绝**
**如此的做法,也已经成为现代C++的编程规范之一了**

*那么, 之前的C++是如何处理这样的问题呢 ?*

这种问题还是会出现的, 我们还是之前的态度: 我们不能假定用户并不会犯错, 所以这种情况也应该处理掉的.
那么我们的入手点在于: compiler-gengrated functions 都是默认 `public & inline`
我们的入手点便是: 将其不能直接使用 => **使用访问权限的控制手段**

我们提供以下两种思路:
1.`private & implemention`
2.`base-class & implemention` (1.的基础上改良)

对于1, 我们的思路很明确, 就是使用`private`访权控制来拒绝我们不想使用的成员函数

但是, `member function`和`friend function`会打扰我们.

**于是, 我们就给予它便不实现, implmention => 链接器会报错**

> 链接器报错 link-error
> 上面说到的链接器报错,是这个原因:
> 按照常人理解, 一般没有实现的函数, 会直接编译错误对吧 ?
> 但是C++因为分离式编译的原因 (区别参考于Java) ,前置声明了解一下, pimpl手法基础
> 知道link期间才会报错, 找不到函数的引用, 也就是实现之处, "no reference to FUNC"

于是方法1 可以比较好的解决这个问题了.

尽管链接器报错, 也不会影响运行时效率, 可是我们还想尽早的报错为好, 将错误提前到编译期(Compiler)
有办法吗? 一定是有的!

使用OOP的手法: 继承
我们可以构造一个基类(因为不给予实现),将之继承, 然后正常编写即可.
**那么为什么会将报错提前至编译期呢 ? **
> 无法进行构造, 因基类没有构造函数, 就是说一定要是用这个类构建对象, 才能错误提前到编译期

举个例子如下:
```cpp
class noncopy {
private:
    noncopy(const noncopy &);
    noncopy &operator=(const noncopy &);
};

class Temp : public noncopy {
....
};

Temp t;
```
这便是正确的使用举例

**我们建议使用现代C++语义明确的手法, C++03, 乃至98的手法, 只是作为开阔思路的解法了解即可**

> 请记住
>
> 为驳回编译器自动(暗自)提供的机能, 可将相应的成员函数声明为privat并且不予实现.
> 使用像uncopyable这样的base class也是一种做法

## Item 07: Declare destructors virtual in polymorphic base class

如果要在C++中提到 `virtual`这个词, 经常用在这两个方面: `virtual继承` 与 `virtual function`

我们这一部分专注于`virtual function`的内容, 虚继承的内容都是后话.

首先来看一个例子:
```cpp
class base {
public:
    base();
    ~base();
};

class Aderive : public base {...};
class Bderive : public base {...};
class Cderive : public base {...};

base *pb = get_derive_pointer();   // 为了多态需求使用, 基类指针(/引用皆可)

delete pb;
```

`delete pb;`**会造成严重的灾难性后果 !**
**因为delete一个基类指针, 而且它的析构函数是non-virtual, 在派生类中, 此析构仅仅是base析构**
**这样, 你也能想到析构的下场了吧, derive部分无法析构, 且指针销毁, 只剩下一个不完全的对象**

那么,正确的做法呢?

**使用virtual-destructor 在base-class中 (子类继承后,virtual属性一并继承)**

这个其实是多态性的基础--C++中,基类指针/引用可以指向派生类的对象, 因此才能实现多态性

这样一来, 就可以舒服的解决这个问题了.

*那么, 从一个方面来考虑, 未提供nvirtual destructor的函数, 就很麻烦了, 万一被继承就要出篓子*
*比如: std::stirng, std::iostream等*
*EffectiveC++的作者在感慨 C++没有的 final 已经在现代C++中加入, 但是我在< string >中还没找到*
*反正, 注意对于未提供virtual destrutor的class, 可能本来就是没有被设计作为多态base-class考虑*

我们再来稍微说说vitual的事,

下面这些是, hepangda给我的科普,

> C++中实现的多态是成本最小的之一. 很简单的问题, 使用计算机时, 多态是我们目前不可预测的,
> 在编译期无法预知, 那么最简单的, 就是打表的思想, 我把可能的情况都塞进去, 到时查表就行
> 而具有Basic Object的语言, 就可以完全实现反射, Java, Ruby等...

基于上面的这段话: 
virtual实际上对class进行了打表, 将可能的函数指针封进表中, 成为虚表 (virtual table)
而每个class中会留一个隐藏的虚表指针vptr (virtual ptr)
使用virtual会**大程度的提高开销 ! ! !**, 他会使可执行文件增加容量

举一个EffectiveC++中的例子:
(64位环境)一个类中,有两个int, 应该占8个字节, 这个时候加上一个virtual, 多一个vptr
class体积增加100% !

```cpp
class A {
    int a;
    int b;
};        => 8 Bytes

class B {
public:
    virtual ~B();
private:
    int a;
	int b;
};        => 16 Bytes
```

另外我们再来说说**抽象基类**的事情:
abstract class 指的是含纯虚函数的类,
*很不幸, 我们因为没有interface的关键字, 导致定义接口要使用抽象基类的手法*
**我们要讨论的是, 如何拥有一个抽象基类**

**其实,也不麻烦,就是标题的 纯虚析构函数 即可**
*因为纯虚 (pure virtaul), 所以是抽象基类*
*同时又是 virtual 析构函数, 所以不用担心析构的问题*

**这里的窍门是: 要为 pure virtual destructor 提供一份定义**
**否则, derive class 的destructor都会爆炸的, 具体上一条款中关于implemention的问题**

```cpp
class Temp {
public:
    vitrual ~Temp();
}

Temp::~Temp() { ; }
```

因此, 给出我们使用virtual的建议:
**记住: 仅仅在有多态需求时使用virtual函数, 其余时候(仅作为基类, 一般类)能不用就不要用virtual**

> 请记住
>
> polymorphic(带多态性质的) base classes 应该声明一个virtual 析构函数. 如果class带有任何virtual函数, 它就应该拥有一个virtual析构函数
>
> 不是设计用来作为 base classes（基类）或不是设计用于 polymorphically（多态）的 classes（类）就不应该声明 virtual destructor（虚拟析构函数）

## Item 08: Prevent exception from leaving destructors

异常可谓是进行OOP中所必不可少的。
从某个方面来说, 使用异常机制可以改变程序设计的模式, 使我们更加专注于逻辑设计
将平时繁琐的错误处理情况, 按死在异常中(简单地说, 不容易分心)

C中也实现了近似于异常的机制, 起源于goto的语法, `setjump/longjump`
Java中的异常机制十分完善, 并且保证了足够的效率.
反观, C++中的异常机制, 因为是后来加入的体系, 所以在性能和效率上都差强人意.

一个不得不承认的事实: 
**C++中应该尽可能少, 甚至不使用异常, 而且我们一般对于异常的处理是杀死**

那么, 我们在这一条款中, 到底要讨论什么主题呢？

*我们对于异常机制的使用, 以及关于OOP设计接口的一部分讨论*

对于异常的处理, 我们以下面的例子来谈：
```cpp
class SQLaffair {
public:
    SQLaffir();
    ~SQLaffair();
private:
    SQL *psql;
};
```

我们使用指针作为成员, 这也是pimpl手法的应用之一。

我们重点的是要讨论在析构函数中, 如果发生了异常该怎么办？
我们一般具有两种手段来处理:
1. 悄悄掩盖住错误
2. 让他去死 (PS: Let it die挺好玩的)

### 悄悄掩盖住错误

这一种处理方法, 在某种程度上是为了软件稳定的妥协。
问题并非是毁灭级的失败, 且稳定性更为重要的时候, 我们便放弃对错误的处理, 悄悄掩盖起来

我们为什么会选择悄悄处理错误这样一种决策呢?

*因为在析构函数中抛出异常意味着 "中断程序" 和 "未定义的行为"*

即使悄悄吞下错误, 在一定程度上也比引发上面两种行为, 好得多。

### 让他去死

这是一种大多数情况的策略

```cpp
SQLaffir::~SQLaffir()
{
    try {
        std::runtime("msg");
    } catch(std::runtime re) {
        std::abort();
    }
}
```

但是, 这种策略也显得过于粗暴, 用户在被强制中断, 不太友好。

所以, 我们的建议是, 开放一个接口给用户使用, 同时析构进行调用。

如下：
```cpp
SQLaddair::close = flase;  (TYPE: std::atomic<bool> close;)
SQLaffir::close()
{
    SQLptr.close();
    close = true;
}

SQLaffair::~SQLaffair()
{
    if (!close) {
        try {
            SQLptr.close();
        } catch(...) {
            // 两种策略
        }
    }
}
```

上面的策略便是我们推崇的真正应该使用的处理手法。

为什么这样呢？因为他有如下优点:
1. 析构函数中严格的使用"悄悄吞掉" 或 "让他去死"的策略, 避免抛出异常
2. 我们给了用户一个接口, close() 来处理异常情况

EffC++中解释了, 使用close()与接口设计的矛盾性：
虽然从设计上来讲, 不应该将资源管理权交给用户, 因为不安全。

**但是, 我们在此处将资源控制权交给用户, 为什么?**

因为,关于SQL连接关闭是不可轻易吞下的错误, 我们应该给予用户以控制并处理错误的权利
所以, 此处我们先将问题暴露给用户, **在用户放弃处理的时候, 我们在析构函数中, 使用两种策略处理**

综上所述:
**1. 不能让异常在析构函数中抛出, 会导致未定义行为/crash, 应该选择悄悄吞下/让他去死**
**2. 基于1, 我们应该作出, 必需要处理的异常, 在析构函数之前进行处理, 别让异常等到析构, 往出传播**

*EffectiveC++中, 候捷先生的翻译是: 别让异常逃离析构函数*
**私以为更妥当的翻译应该是: 在析构结束之前处理错误, 不要让异常在析构函数中抛出**

> 请记住
>
> destructor（析构函数）应该永不引发 exceptions（异常）。如果 destructor（析构函数）调用了可能抛出异常的函数，destructor（析构函数）应该捕捉所有异常，然后抑制它们或者终止程序。
>
> 如果 class（类）客户需要能对一个操作抛出的 exceptions（异常）做出回应，则那个 class（类）应该提供一个常规的函数（也就是说，non-destructor（非析构函数））来完成这个操作。

## Item 09: Never call virtual function during Constructor or Destructor

继承是面向对象编程中很重要的一个特性.

设计并构建功能完整并强大的继承体系是一项复杂的工程.

在这其中我们会翻一个很经典的错误: **在构造/析构函数中使用virtual函数**
乍看之下, 是不是还是挺有道理的.

**事实上, 这是一个严重的错误: 因为在析构, 构造期间. 类并非完整的, RTTI并不能得到派生类类型**
**因此, 我们所调用的只能是基类版本, 会导致严重的错误! 并不能达到我们的需求. 严重性可想而知.**

来看这样一个例子:
```cpp
class base {
public:
    base();
	~base();
	virtual log() const = 0;
};

base::base()
{
    log();
}

class Aderive : public base {
public:
    Aderive();
	~Aderive();
};

class Bderive : public base {
public:
    Bderive();
	~Bderive();
};

Aderive ad;
```

上面的示例, 我们构造ad对象, 构造过程会顺利完成.
*但是, 其中使用的log(), 是base::log(), 不得不承认, 这是个令人惊奇的事实.*

**原因正如上面所说: **
*基类构造发生在派生类部分构造之前(base part construct before derive part)*
*我们使用vitual函数, RTTI得到的是base类型, 而非是derive类型, 不能满足我们的要求*
*更通俗的来讲: 此时的virtual并非是virtual, 仅仅是它自己的版本*

*正是因为, derive class尚未初始化完成, 使用派生类中的内容, 可能导致不安全行为/未定义行为*
*所以, 不允许virtual, 仅仅能使用base::log()版本.derive construct时, virtual不会下降到derive class*

**准确的理由: 此时RTTI结果是base类型,dynamic_cast (此时进入base部分, derive部分视为未定义)**
**析构函数也是同理的.**

*事实上, 防止构造/析构过程中出现virtual调用时很难检测的错误*
**因为, 提供了实现版本, 就会出现正常运行, 错误结果的情况.**

*注: pure virtual的版本可以通过编译, 调用会凉凉...*
```
pure virtual method called
terminate called without an active exception
```

例如:
```cpp
#include <iostream>
using std::cout;
using std::endl;

class base {
public:
    base() {init();};
    ~base(){};
private:
    void init() {log();};
    virtual void log() const = 0;                   // pure virtual 运行, ERROR
	// virtual void log() const {cout << "base";}     // Result: base
};

class derive : public base {
public:
    derive(){};
    ~derive(){};
private: 
    virtual void log() const {cout << "derive";}
};

int main(void)
{
    derive b;

    return 0;
}
```

*明白了吗? 即是说, 一旦是impure virtual 就是很难处理的Bug*

那么, 有没有可以比较好的解决办法呢? **派生类构造向基类传递参数**

```cpp
#include <string>

class Transaction {
public: 
    explicit Transaction(const std::string &info);
    void logTransaction(const std::string &info) const);

    ...
};

Transaction::Transaction(const std::string &info)
{
    logTransaction(info);
}

class BuyTranscation : public Transaction {
public: 
    BuyTranscation (parameters)
        : Transaction(createLogInfo(parameters))     // 使用传参的形式完成原多态需求
    { ... }
private: 
    static std::string createLogInfo(parameters);   // static保证此部分与对象无关, 无危险
};
```

综合所述:
**不要在构造/析构中使用virtual, 因为这时刻他们的类型是base, RTTI结果亦是如此**

*或者说, 此时刻, 不会从base下降到derive层次*

> 请记住:
>
> 在 construction（构造）或 destruction（析构）期间不要调用 virtual functions（虚拟函数），因为这样的调用不会转到比当前执行的 constructor（构造函数）或 destructor（析构函数）所属的 class（类）更深层的 derived class（派生类）。

## Item 10: Have assignment operators return a reference to *this

此部分的内容实际上较为简单, 主要是阐述了一个关于`operator overload`的默认约定

即:**运算符重载应该返回*this的引用**

我们可以看几个例子:

```
a = Obj1 + Obj2;
a = getVal();
```
为什么可以这样写, 因为a是左值, **返回对*this的引用, 便能保证是左值**
**若返回并非是左值, 则第二次运算会失败, 因为a仅仅是一个右值**

简单地说: **我们返回非*this引用也无大碍, 但是,为了与用户习惯以及内置类型行为近似**
**我们强烈建议, 运算符重载返回对*this的引用.**

*PS: 这只是建议, 并非是编程规范*

> 请记住:
>
> 让 assignment operators（赋值运算符）返回一个 reference to *this（引向 *this 的引用）。

## Item 11: Handle assignment to self in operator=

如题所示, 这一条款我们主要讨论的是: 在`operator=`中处理"自赋值"情况

~~这里再喷一下候捷先生的翻译, 不想多说~~

自赋值的问题, 怎么回事?

先来看一个正常的operator=

```cpp
class Temp {
public:
    const Temp &operator=(const Temp &rhs);
private:
    int *p;
}

const Temp &Temp::operator=(const Object &rhs)
{
    delete this->p;
	this->p = rhs.p;
	
	return *this;
}
```

这其中会造成什么问题吗? **是的, 当自赋值时, 这样的运算符重载会产生未定义的行为, 因为悬垂指针**

*换种思路, 我们再来分析一下, 这种"自赋值"常见吗?*

*不幸的是, 基于两方面的原因. "自赋值"是一个不好规避的问题*
*1. 我们不能假设(assume)用户的行为, ~~把他们当成蠢蛋就行了~~*
*2. 我们更多的遇到的情况是, "隐式"自赋值*

例如: 
```cpp
a[i] = a[j];

int *p = ...;
int &a = ...;
*p = a;
```

*上面的例子便是两种常见的"隐式"自赋值情况, 需要十分的当心*

扯了这么多, 我们就来说说**真正处理自赋值的方法**

### 使用if-else控制流

```cpp
class Temp {
public:
    const Temp &operator=(const Temp &rhs);
private:
    int *p;
}

const Temp &Temp::operator=(const Object &rhs)
{
    if (*this == rhs)
	    return *this;
    
	delete this->p;
	this->p = rhs.p;
	
	return *this;
}
```

优点: 一旦遇到自赋值, 可以直接退出处理情况.
缺点: 增加了控制流, 开销不是那么容易抹掉的

### 使用Copy and Swap ("CAS")

也可以简称为CAS, 不过与那个大名鼎鼎的CAS是完全不同的东西.

```cpp
class Temp {
public:
    const Temp &operator=(const Temp &rhs);
	void swap(const Temp &lhs, const Temp &rhs);
private:
    int *p;
}

void Temp::swap(const Temp &rhs, const Temp &rhs) {...}

const Temp &Temp::operator=(const Object &rhs)
{
    Temp t(ths);
	swap(*this, t);
	
	return *this;
}
```

优点: 可以安全的处理任何情况
缺点: 当自赋值可能性比较大时, 效率不如增加控制流的手段.

*这两种手法, 可以比较好的处理自赋值, 选择那种,我们要根据情况来判断*
*另外, 禁止自赋值行为时, assert()断言就派上用场了*
`assert(*this == ths);`

> 请记住:
>
> 当一个 object（对象）被赋值给自己的时候，确保 operator= 是行为良好的。技巧包括比较 source（源）和 target objects（目标对象）的地址，关注语句顺序，和 copy-and-swap。
>
> 如果两个或更多 objects（对象）相同，确保任何操作多于一个 object（对象）的函数行为正确。

## Item 12: Copy all parts of an object

拷贝时勿忘掉任何一个对象成分

这是一句缄言, 拷贝赋值的时候, 缺少成分是最忌讳的

那么, 我们要注意的是什么情况呢?

主要有这样两种情况:
**1. 局部成分**
**2. 继承成分**

### 1. 局部部分

局部部分, 主要考虑的是这样的情况: 后续进行class修改时, 有的局部成分被忘记和省略

那么唯一的办法: **自己相信斟酌, 确认将所有成分进行拷贝**

### 2. 继承成分

相对于局部部分, 继承成分乍看之下会很复杂: **因为我们可能使用复杂的继承链**

但是, 继承也有继承的好处.

**只要我们保证层层严格使用基类的构造函数/拷贝赋值, 就可以保证每一部分不会被遗漏**

```cpp
{
    ...
    Base::operator=(rhs);
    ...
}
```

**一个重要问题: 不要试图用构造函数和拷贝构造相互实现, 没有意义! (最多使用私有方法简化重复工作)**

> 请记住:
>
> copying functions（拷贝函数）应该保证拷贝一个 object（对象）的所有 data members（数据成员）以及所有的 base class parts（基类构件）。
>
> 不要试图依据一个 copying functions（拷贝函数）实现另一个。作为代替，将通用功能放入一个供双方调用的第三方函数。
