---
layout: post
title: <星月夜> Designs and Declarations
date: September 26, 2018 7:30 PM
excerpt: Chapter IV 设计与声明. 其实这是一个复杂的问题. 我们在这里谈一些基本, 核心的设计准则. 其中涵盖了Class的设计与声明手法.
categories:
- C/C++
tags:
- EffectiveC++
toc: true
comments: true
---

*完成一个大型项目, 都是从设计开始. 设计可以很复杂, 也可以很粗糙. 但是为了程序的可维护性和发展性*
*进行一定程度上的核心设计十分关键, 我们本篇便来谈谈设计的艺术和一些技巧*

## Item 18: Make interfaces easy to use correctly and hard to use incorrectly

大道至简 -> `使得接口容易被正确使用, 不易被误用`

那么, 我们怎么才能做到**容易使用**这一设计准则呢?
~~其实, 在EffC++中也没有给出明确, 切实, 有建设性的建议~~

让我自己来说: *更多的从每个用户的角度出发, 让他们能够安心使用正确的接口, 便是至上之道*

下面有几个比较有启发的例子, 我们可以来评价一下:

### Type System (类型系统)

先来看一个日期类, 它需要进行日期构造.
```cpp
class Date {
public:
    Date(const int &year, const int &month, const int &day);   // base on item 20
    ~Date();
private:
    int year;
    int month;
    int day;
};
```
其实上面就有一个显而易见的问题: `Date(...)`构造时参数的问题
我们传参的顺序造成的影响? 传参的参数范围如何限制? ...etc

这个时候, Type System其实是一个比较方便的手段.
**我们, 可以使用类型来显示要求用户传参内容, 及顺序**

```cpp
class Date {
public:
    struct Year {
    public:
	    explicit Year();
        ...
	};
    struct Month {
    public:
        explicit Month();
        ...
	};
	struct Day {
    public:
        explicit Day();
        ...
	};
    Date(const Year &year, const Month &month, const Day &day);   // base on item 20
    ~Date();
private:
    int year;
    int month;
    int day;
};

// Using Class -- Date
Date d(Year(2011), Month(3), Day(14));     // Date: 2011,3,14
```

**类型系统**已经在此处显示出它的作用了, 但是, 还不够.
*万一, 超出范围的内容, Out of Ranage, 那就还是凉了...*

那么, 我们就可以拿出OOP的杀器之一: 封装

我们的思路是: 将其封装, 仅仅允许其返回指定范围的结果
比如:

```cpp
// 封装month的示例
class Month {
public:
    static const Month Jan() { return Month(1); }
    static const Month Feb() { return Month(2); }
	...
private:
    Month(int month) {
        this->month = month;
    };
    int month;
};

// Using class Month
Date(Year(2011), Month::Mar(), Day(13));    // 防止Month取值越界
```

### std::shared_ptr< T >

这一部分,其实是RAII的一部分, 其中提到了使用智能指针的技术来管理资源.
**实际上, 直接RAII来的更方便高效**

*此处, 是为了思想上认识到,接口的易用性设计,所以把智能指针单独拿出来说*

```cpp
widget *getWidgetPointer();
Widget *pw = getWidgetPointer();

std::shared_ptr<Widget> getWidgetPointere();
auto pw = getWidgetPointer();                // remove the risk of memory leak at root
```
**与其返回handle, 不如直接给它一个智能指针, 可以直接解决掉资源泄漏的问题**

> 作为题外话, 我们再来聊聊智能指针(std::shared_ptr< T >)
>
> 智能指针, 首先它在实现上, 体积比较大. 不如原生指针来的痛快
> 同时, 它其中还使用了动态内存进行资源管理
> 另外, 智能指针在多线程下的表现也比较一般(尤其是内部计数)
> 但是, 它所带来的编程安全的作用, 证明我们牺牲部分效率是值得的.

*当然, 我们这里的重心不在智能指针上, 只是智能指针 对于接口设计的提升, 以及防止误用上, 作用很明显*

> 请记住 :
>
> 1. 好的接口很容易被正确使用, 不容易被误用. 你应该在你的所有接口中努力达成这些性质
> 2. "促进正确使用"的方法包括接口的一致性, 以及内置类型的行为兼容
> 3. "阻止误用"的方法包括建立新的类型, 限制类行上的操作, 束缚对象值, 
> 以及消除客户的资源管理责任
> 4. tr1::shared_ptr(已纳入标准库 std::shared_ptr)支持定制型删除器, 这可防范DLL问题,
>  可被用来自动解除互斥锁,(不就是RAII么, \抠鼻);

## Item 19: Threat class design as type design

*说的没错, 为了设计出高效, 简洁,易用的类, 我们应该将其当作type来设计*
~~但是, 全部考虑, 基本上是不可能. 我们一般涉及的class, 能符合其中核心的要求就很可以了~~

*EfectiveC++中提供了完整的内容,  参考这进行对比即可*

> 请记住 :
>
> Class的设计就是Type的设计. 在定义一个新的Type之前, 
> 请确定你已经考虑过被条款覆盖的所有主题

## Item 20: Prefer pass-by-reference-to-const to pass-by-value

现代C++提升效率的很重要的手法之一就是: **减少拷贝**
为了实现这一点, 有了下面这样的编程技巧:
- 构造函数初始化列表

- 传引用调用

- 移动语义

- ...

这其中, 其实用引用传参来替换值传参, 
即pass-by-reference-to-const => pass-by-value
起到了十分明显的作用

传值是C++从C继承而来的.但是, 随着程序设计的体量越来越大, 系统越来越复杂.
**不假思索的传值, 已经是丧尽天良的了**
因此, 我们的建议是 `pass-by-reference-to-const`

思考下面的例子:
```cpp
class Base {
public:
    Base();
private:
    std::string Bstr;
    std::string Bsstr;
...
};

class Derive : public Base {
public:
    Derive();
private:
    std::string Dstr;
    std::string Dsstr;
};

bool isDerive(Derive d);    // think this function ?
```

思考上面的函数调用. 是不是觉得挺正常, 那么让我来给你分析一下:

1.调用Base构造函数,
2.Base内部还有 Bstr, Bsstr两个string的构造
3.调用Derive的构造函数
4.Derive内部还有 Dstr, Dsstr两个string的构造

总结一下: **共使用了6个构造函数**
*而且, 根据~~函数名我们可以推断~~, 这只是一个一般的判断函数, 过程很短, 这样大费周章的传参数*
*这一切值得吗? (引自 < 冰汽时代 > -- 11 bits studio)*

显然, 你自己也觉察到了: **不值得, 这便是使得C++效率降低的理由之一**

那么, 有什么办法吗? *嘿嘿嘿, 求我,我就告诉你*

~~(就当你求过我了)~~

**pass-by-reference-to-const**

这便是解决效率问题的一把杀手锏. 回到刚才的问题上, 如果这样调用函数:
`bool isDerive(const Derive &d);`

传参基本上是没有开销的 ("基本上"的理由, 我们后面会讨论)

不过性能提升是因为使用reference, (不懂reference的自行面壁)

### To use pass-by-reference-to-const

如何使用 `pass-by-reference-to-const` ?

简单地说: 将原`pass-by-value`替换为`pass-by-reference-to-const`即可

想过没有, 为什么要用const ?

**使用传值的手法, 我们作出的修改并不会反映到原数据上, 只是处理它的拷贝**
**但是, 使用引用, 所作出的修改是在同一数据上, 所以, 为了说明不会进行修改, 我们使用const限定**

万一真的要修改呢? 想想我们的"万能引用" (C++ 11起), 即右值引用 + 引用折叠

即:

```cpp
(pass-by-reference-to-const) => const T &     // 不修改参数时

(pass-by-R_reference) => T &&                  // 修改参数时
```

### the cost of pass-by-reference

上面说过, 基本没有开销, **基本上**, 这就是一个可以商榷的修饰词了

实际上, 因为目前而言, **引用的底层实现基本上还是指针**,
所以, 引用传参, 就是当时C中的, "传指针"行为, 它准确的传递了数据的地址

所以, 它基本上没有开销, 开销多大呢? 一个指针的大小.

```cpp
// Kangkang拿这个搞过我

struct A {
    int &a;
};

sizeof(struct A) ? => result: 8 (64-bit)
```

*在可以憧憬的未来. 一旦硬件级别真正定义了引用, 那么它的开销就是 0, 就是真正意义上的别名, 无开销*

### Deal with slice down object

我们之前其实讨论过切割(slice down)的问题, 指的是, 派生类构造不完全, 仅仅只有基类部分
派生部分被切割掉.

那么, 在传值时, 很容易(基本上一定)发生slice的问题.
**比如, 为了多态需求, 接受形参为基类对象(派生对象)时, 传递了派生对象**

**就会导致在func内部仅仅只有Base Class的部分, 无法实现多态需求**

解法就是: 使用基类指针/引用, 我们这里主要推荐的是基类引用

`bool do_something(Base obj)`
=>
`bool do_something(Base *obj) / bool do_something(Base &obj); [const 根据需求]`

### Have to use pass-by-reference-to-const ?

虽然, 上面说的那么愉快, 我们可以将所有的情况规划到引用以内 (C++11 的右值引用)

那么, 一定使用引用吗 ? (~~虽然, 某J语言就是这么干的~~)
我们可以举出一个广泛使用传值的例子 --**STL(迭代器+函数对象)及内置类型**

这便是我们的反例, 在这两方面, 传值的手法广泛使用.
我们的理由如下:

1.STL的广泛使用显而易见, 我们应该与其保持一致, 且迭代器, 函数对象目前实现也为指针
2.内置类型的传值, 在目前还是指针实现引用的时候, 不一定就效率低

比如说: `int a`与`int &b`, 32/64位大小上是一致的. `char`, 就是个更明显的例子了.
**所以, 内置类型(或自定义小类型)传值有的时候效率会更高**

但是, 反对小类型传值, 也有这么两个理由:
1.单个内置类型开销只比引用小一点, 但是一个自定义类型中有多个内置类型时, copy成本直线上升
2.我们不能假定Type的实现是一成不变的, 如果实现有变化, 我们的代码需要大幅翻新 (作者下笔时:
  有的string实现就比一般版本大7倍)

所以, 我们的结论是: **除了STL(迭代器 + 函数对象)和内置类型之外, 最好遵守我们的准则**

> 请记住 :
>
> 1. 尽量以 pass-by-reference-to-const 替换 pass-by-value 
> 前者通常比较高效, 且能避免切割问题
> 2. 以上规则并不适用于内置类型, 以及STL的迭代器和函数对象. 
> 对他们而言, pass-by-value往往比较恰当

## Item 21: Don't try to return a reference when you must return a object

我们完全什么要说明这一条款呢 ?
*因为经历了Item 20的洗礼, 你完全有可能任何时候都想返回引用, 事实上可取不可取呢?*

**答案显然是否定的**

在我们需要用引用返回对象时, 我们是不能假定已经存在我们需要的对象的
所以, 我们就需要在函数内创建这样一个对象, 方法有二: stack / heap

### stack object

```cpp
// stack object

const int &stack_object
{
    int a = get_value();
	return a;              // a allocated on stack
}
```

上面的例子便是分配在栈上的对象, 返回reference.
**我们可以明确地说: 这就是导致程序错误的起点 !**

**在栈上分配的对象, 退出块之后便已经销毁. 所以这个引用已经失效了**

### heap object

```cpp
// heap object

const int &heap_object
{
    int *a = new int(4);
	
    return *a;            // a allocated on heap
}
```

看着很美好, 没错吧 ? 
假象, 试问: 现在如何释放, 这个分配在heap上的内存 ? 

**我们现在是无法得到, 隐藏在reference背后的指针的, 因此造成内存泄漏**
返回的是指针指向的内存, 我们无法得到指针的地址.取地址也并非指针地址(而是指向内存的地址);

### static object

即使到了这一步, "有毅力"的人还不放弃,
想要试试static 对象. 是的, 没错.可以尝试, 但是来看看下面的例子

```cpp
int &getValue()
{
    static int a;
    a = setValue();

    return a;
}

// Using function

m = getValue();
n = getValue();

m == n ? => // while be true always
```

这个简单的例子上, m, n已经是永久都相等了. 那么, 要是放在operator上, 后果可想而知.

至此, 甚至还想考虑static数组等等措施, 都是不切实际的 (数组大小, 如何存放 ? )

总结: 
*虽然析构, 构造的成本会影响我们的效率. 但是, 那是构建在程序正常运行的基础上*
*一旦我们的程序因为固执的返回引用而引发一系列错误的时候, 就显得得不偿失了*

**所以, 不要畏惧pass-by-value, 如果它能带给我们正确的行为, 析构/构造的开销就交给编译器去优化了**

> 请记住 :
>
> 绝不要返回 pointer 或 rerference指向一个 local stack 对象, 
> 或返回 reference 指向一个 heap-allocated 对象, 
> 或返回 pointer/reference 指向一个 local static 对象而有可能同时需要多个这样的对象. 
> 条款4 已经为 "在单线程环境中合理返回 reference 指向一个 local static 对象"
> 提供了一份设计实例

## Item 22: Declare data member private

我们在使用class的时候, 接触到了三种访问权限标识符:`public`, `protected`, `private`

> 三种访问权限关键字如何使用呢 ?
>
> public: 公共的, 是类给外部开放的接口, 供其他用户调用
> protected: 受保护的, 这个是给Derive Class在继承体系中使用的, 有着"内部public"的感觉
> private: 私有的, 为了数据封装, 属于类的私有内容. 即使是派生类也无权访问

那么, 到底何为封装呢 ? 封装的意义又在哪里呢 ?

从OOP的设计角度而言, **封装就是为了屏蔽, 提高程序设计的粒度**

我们可以去使用封装的数据, 而不用关注起底层的实现机制. 即使底层的实现完全重写, 用户也不会意识到

合理的设计API的目的也在于此, 进行底层内容的替换不会导致客户代码被大量重写

封装能够做到: **安全性, 便利性, 可维护性, 可扩展性**

问题来了: 如何做到合理的封装 ?
**简单地一句话: C++中, private具有封装性 和 其他没有封装**

### private & protected

`private`是我们常用的访权控制符, 除了member-func, friend-func之外,都不能进行访问.

那么, `protected`呢 ? 这个其实是被我们经常误解的一部分

`protected`表面上看上去, 并不能被外部访问, 看上去是具有封装性的, ~~假象~~
`protected`的内容,在Derive Class中是可以被随意访问的, 
也就意味着: 修改底层, 同样会导致**大量客户码重写**

所以, 我们说**private具有封装性, 其他的都没有封装性**

> 请记住 :
>
> 1. 切勿将成员变量声明为private. 这可赋予客户访问数据的一致性, 可细微划分访问控制, 
> 允许约束条件来获得保证, 并提供class作者以充分的实现弹性
> 2. protected并不比public更具有封装性

## Item 23: Prefer non-member non-friend functions to memeber functions

上一条款我们提到了封装性. 这一条款继续延伸封装性的话题.

~~EffC++使用一个Browser举例, 我就大言不惭的用我的WebServer举例了~~

```cpp
// socket.h
namespace PlatinumServer {
class socket {
public:
    socket() = default;
    ~socket();
    void connect();
    ...
private:
    int sock_fd;
    struct sockaddr_in sock_sockaddr;
    bool bind();
    bool listen();
    ...
};
}

// socket.cc
namespcae PlatinumServer {
void socket::connect()
{
    int buf = 1;
    setsockopt(this->sock_fd, SOL_SOCKET, SO_REUSEPORT, &buf, sizeof(int));

    if (!bind())
        err_handle("bind");
    if (!listen())
        err_handle("listen");
}
}
```

我们来分析一下上述代码: 
将`PlatinumServer::socket::bind()`, `PlatinumServer::socket::listen()` Wrapper进入`PlatinumServer::socket::connect()`中
简单地的来说: 就是封装.

**事实上真的是这样吗? 答案是否定的**
这是我们曲解了OOP封装设计所导致的.

所谓封装: 指的是,将底层细节包裹尽可能深, 不被外接所侦测和访问.
**但是, 我们上面的做法, 无疑增加了对于封装内容的访问**

一个粗糙的指标便是: 通过可以访问的函数数目来判定其封装程度.
那么, 显而易见的就是: member. friend函数会访问我们封装的数据, 因此而降低我们的封装性

结论就是: 尽量以`non-member & non-friend`来替代`member function`

比如: 我们需要这样来实现`PlatinumServer::socket::connect()`

```cpp
// socket.cc
namespace PlatinumServer {
void connect(socket &sock)
{
    int buf = 1;
    setsockopt(this->sock_fd, SOL_SOCKET, SO_REUSEPORT, &buf, sizeof(int));

    if (!sock.bind())
        err_handle("bind");
    if (!sock.listen())
        err_handle("listen");
}
}
```
所以, 我们倾向于设计`PlatinumServer::connect()`而非`PlatinumServer::socket::connect()`

这里有两个要点:
1. 我们在`member-fucntion`的对立面,应该是`non-member & non-friend`, 而不是说,非成员  一定是友元,
 这样并没有提高我们的封装程度. (下一条款, 我们会讨论成员与非成员的选择)
2. 一个比较重要的点: `connect()`不应该被设计为成员函数, 而应该是非成员&非友元函数, 
 并不意味着它不能成为其他类的函数 (~~对Java和C井开发者而言~~),
 而我们在C++中的惯用做法是: **将此函数放在与`PlatinumServer::socket`相同的命名空间**
 正如我们上面所做的一样.

### namespace | Class

namespace(命名空间)与Class(类)并非是针锋相对的关系.
通过合理使用这两种工具, 我们可以减少名字冲突, 并且更好地实现封装的需求

namespace: 跨文件进行封装, 在避免名字冲突上尤其方便好用
class: 可以构造出相当规整的继承体系链来

但是他们也有不同之处: namespace构造出来的是平行的关系, 而class构造的是立体的架构

比如: 我们常使用的 C++ Standard Library, 并非是一个 < C++StandardLibrary >吧
他将我们要应用的不同部分, 划分到了,不同的头文件
e.g. < vector >, < list >, < iostream >等

反观, class就不能用在此处. 因为这些不同的标准库设施, 并没有实际的继承体系

合理的使用namespace/class 这些类设计工具, 会使我们事半功倍

> 请记住 :
>
> 宁可拿non-member & non-friend 函数替换member函数.
> 这样做可以增加封装性, 包裹弹性和机能扩充性

## Item 24: Declare non-member functions when type conversions should apply to all parameters

在聊聊这一条款之前, 我们先来看一个关键字 `ecplicit`

> explicit
>
> 英语释义: explicit
> adj.明确的，清楚的;直言的;详述的;不隐瞒的
>
> 这个关键字是C++, 限定在构造函数上使用的, 它表示 **不允许隐式类型转换**
> 默认的构造函数是 non-explicit的

具体是什么意思呢?

不允许进行隐式类型转换/复制初始化

例如: `vector`大多数构造函数都是explicit的, 会产生隐式类型转换, 造成意义不明
`string`大多数构造函数都是non-explicit的, 可以完成隐式类型转换

我们这一条款的重心来了: 如果参数都需要类型转换的时候, 怎么办呢?

总是说运算符重载, 我们就来看看运算符重载的例子:
```cpp
class Temp {
public:
    Temp(int a = 0, int b = 0) {
        this->a = a;
        this->b = b;
	}
    ~Temp();
    Temp operator+(Temp &rhs);
    int getA();
    int getB();
private:
    int a;
    int b;
}

Temp Temp::operator+(Temp &rhs)
{
    return Temp(this->a + rhs.getA(), this->b + rhs.getB());
}
```

我们常见的, 运算符重载的两种形式之一: 成员函数(另一种为: 友元函数)
那么, 下面的场景, 它应用的怎么样呢?

```cpp
Temp a;
auto m = a + 3;             // Correct
auto n = 3 + a;             // Compiler Error !
```

为什么会这样呢?

```cpp
// 转换一下
auto m = a.operator+(3);
auto n = 3.operator+(a);
```

可以看出, a中是有, `Temp::operator+(...)`此成员函数的, 没有, 自然是的调用失败
而且, 我们在全局范围内, 并不能找到 `Temp operator+(Temp &lhs, Temp &rhs);`函数

*Tips: 其中可能会迷惑, 3并非是Temp类型, 如何进行调用的呢? 想想explicit吧 !*

最后回到我们的重点上来: 如何实现需求呢?
**那就是: 使用non-member function**

```cpp
Temp operator+(Temp &lhs, Temp &rhs)
{
    return Temp(lhs.getA() + rhs.getA(), lhs.getB() + rhs.getB());
}
```
这样便可以很好的解决我们目前遇到的问题了.
(~~也即是说,我们平时实现的习惯是值得商榷的, 因为我们仅进行相同类型之间的运算~~)

那么, 我们的问题来了:
**上一条款着重说明了, **`member function`**相对的是**`non-member & non-friend function`
那么, 我们在这一条款中讨论的, non-member function, 是否也应该是non-friend呢?

在此情景下, 此函数不应该被设计为 `friend`函数, 原因: **我们可以用接口访问私有数据**
**因此应该尽可能避免破坏封装性, 我们应该减少友元的出现**

*因为, 友元和成员, 不可避免的会破坏封装程度*

> 请记住 :
>
> 如果你需要为某个函数的所有参数(包括被this指针所指的那个隐喻参数)进行类型转换, 
> 那么此函数一定是个non-member

## Item 25: Consider support for a non-throwing swap

`std::swap()`是`C++ STL`中一个有意思的玩意. 后来它有多大作用不考虑.

但是, 它在我们平常使用中,是有相当大的便利的.

先来看看它的标准实现, 不出所料, 模板实现
```cpp
template <typename T>
void swap(T &m, T &n)
{
    T Temp(m);
    m = n;
    n = temp;
}
```

这个实现是没有什么意思的, 因为只是进行了对象的赋值和拷贝.(**注意, 一定是可拷贝的, 才能swap**)

现在介绍一种,C++标准库中的手法: [pimpl idiom>>](https://en.wikipedia.org/wiki/Opaque_pointer)

这种手法: 主要是通过不透明指针隐藏类的实现, 同时维护稳定的ABI接口, 以及减少编译依赖的
(~~当然, 没有一种手法是毫无缺点的, 在模板特化, 运行开销, 堆栈开销上, pimpl idiom也有不足)

pimpl idiom (D pointer, 不透明指针), 这不是我们这里讨论的重点

我们在这里要讨论的是, 如果是pimpl手法的实现, `std::swap()`还值得这样做吗?

**肯定是不可行的**, 平白无故多拷贝构造了一个对象, 在本质上只是需要进行指针的交换的时候

现在, 来考虑一下, 有什么合理的手法来处理呢?

### 特化模板

```cpp
class Temp {
public:
    ...
private:
    XXX *impl_idiom;
};

template <>
void swap<Temp>(Temp &a, Temp &b)
{
    swap(a.impl_idiom, b.impl_idiom);
}
```

通常我们不能向std中添加任何玩意, (毕竟标准委员会也不是吃闲饭的, ),但是特化模板是可以被接受的
仍然是,看上去十分美好. ==通过特化模板, 对于指定类型, 直接交换pimpl指针==
但是, 事实上是不能通过编译的
**因为, impl_idiom成员是私有的, 我们不能随意访问它**

我们便萌生出, 添加接口/友元, 两种办法.
明确的可以说: 添加接口, 是在逃避问题, 而且它还可能引发之后出现的一系列新的问题
那么, 友元就可取吗? *也不是不行, 但我们这里一般使用类STL的手法*

### 类STL手法

其实是对上面方法的改进. 我们通过使用`public member-function + 特化template`的方法

```cpp
class Temp {
public:
    ...
    void swap(Temp &rhs) {
        using std::swap;
        swap(impl_idiom, ths.impl_idiom);   // member-func 可访问同类型私有成员
    }
};

namespace std {
template <>
void swap<Temp>(Temp &a, Temp &b) {
    a.swap(b);
}
}
```
这样便是STL通用模式: 类开放公共接口, 模板进行特化调用此接口. 也是为了不适使用friend破坏封装
这个手法能够很好的通过编译, 同时正常的工作

### 偏特化

但是, 需求永远也不能满足.

当我们编写的是类模板, 而不是简单地类时, 问题又出现了. 我们对function template进行偏特化

**标准明确规定, 对类模板可以进行全特化/偏特化, 函数模板不能进行偏特化**
*并非实现问题, 函数模板的偏特化, 一般由函数(模板)重载形式来实现*

```cpp
namespace std {
template<>                                    // function-template偏特化失败
void swap<Temp<T>>(Temp<T> &a, Temp<T> &b)
{
    a.swap(b);
}
}
```

那么, 怎么办呢? ==> 函数重载来完成偏特化的需求

```cpp
// function template overload
namespace std {
template <typename T>
void swap(Temp<T> &a, Temp<T> &b)
{
    a.swap(b);
}
}
```

可行吗? 是的, 这个是可行的. 但是它仍然违背了**我们不建议向std中添加内容**这一准则

还有办法吗? **有的, 限制命名空间**

```cpp
namespace TempStuff {
template <typename T>
class Temp {
    ...
};

...

template <typename T>
void swap(Temp<T> &a, Temp<T> &b)
{
    a.swap(b);
}
}
```
**当然, 现在也就算不上是函数重载了.**

现在, 我们来分析一下, 调用的方法:
```cpp
void doSomething()
{
    Temp a, b;
    using std::swap;
    swap(a, b);
    ...
}
```

```cpp
// 1
class Temp {
public:
    void swap(...);
    ...
};

// 2
namespace std {
templaet <>
void swap<XX>(...) {...}
}

// 3
namespace XX {
template <typename T>
class XX {...}

templaet <typename T>
void swap(XX<T> &a, XX<T> &b) {...}
}
```

> `using std::swap`意义何在 ?
>
> 这是为了配合下面的 `swap(a, b);` 调用的
> 因为C++具有一套复杂的名字查找规则,(==ADL尤其甚之==) [此处暂且不表]
> `swap(a, b);`的调用形式是必须的
> 当我们namespace具有特化的版本(模板类时) [3] / std特化版本(全特化) [2] 时, 优先调用(pimpl_idiom)
> 当没有这些实现时, 我们调用`std::swap`, 来进行一般的交换. [pass-by-value]
> 所以,`using std::swap`的名字曝光吧.

同时, 我们要注意的是, 
`using std::swap; std::swap(a, b);`**的编码是禁止的**
这会使我们前功尽弃, 我们为了提高效率库的特化版本, 并不会被查找, 会直接使用std版本(对象拷贝)
*当然,这里会使用全特化版本 [2], 不过你的类若是模板类, 那么命名空间内的T版本无法被调用 [3]*

现在, 我们合理的梳理一下顺序:

1. 为Class编写public的swap接口成员函数
2-1. 若是class, 则进行std的特化版本`template <> void swap<XX>() {...}` + `XX.swap()`
2-2. 若是class template, 我们便在Class命名空间中, 进行专属swap template编码 + `XX.swap()`
3. 进行swap调用, 核心是 `using std::swap;` + `swap(a, b);`

至此, 我们已经实现了一个基本高效的swap了.
最后一点: **绝对不要抛出Execption**

Emmmm, C++ 11新加入了一个东西, `noexecpt`, 表示不抛出
**因为, 在我们不提示的时候, 编译器默认做好了抛出异常的准备, 会有一些无用功**

而我们明确的指示不抛出, 则可以大幅提高效率.

**而且, 我们进行手动swap编码时, 基本都是pimpl**
**而pimpl是指针, 是内置类型**
**内置类型进行拷贝, 是绝不会抛出异常的的**

**所以, 不抛出异常的swap就显得尤其关键 ! **

> 请记住 :
>
> 1. 当`std::swap`对你的类型效率不高时, 提供一个swap成员函数, 并确定这个函数不抛出异常
> 2. 如果你提供一个`member swap`, 也该提供一个`non-member swap`用来调用前者
> 对于Classes(非template), 需要特化std::swap
> 3. 调用swap时应该针对 swap,使用`using std::swap`声明式, 
> 然后调用swap并且不带有"命名空间资格修饰"
> 4. 为"用户定义类型"进行`std templates`全特化是好的, 
> 但千万不要尝试在std内加入某些对std而言是全新的东西

本篇中, 我们讨论了设计的问题, 下一篇与此篇息息相关, 那便是关于实现(implementations)的内容
