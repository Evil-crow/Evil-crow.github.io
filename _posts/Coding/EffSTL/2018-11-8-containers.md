---
layout: post
title: <STL> Containers
date: November 8, 2018 1:29 PM
excertpt: STL中相当重要的一个组件便是容器. 那么,容器有什么需要注意的点呢?
categories:
- C/C++
tags:
- EffectiveSTL
toc: true
comments: true
---

*第一次听说容器的概念还是在C中, 形容存放对象的对象就叫做容器.*
*当然在C中, 对象与此处的对象不同,广义上的对象,指的是具名分配的内存.* 
*但是容器的概念沿用下来, 在C中数组也被视为一种容器.*
*那么C++中容器究竟是什么样子, 我们又应该注意些什么要点呢?*

首先, 我们来说说容器. **用来存放对象的对象就叫做容器**
在C++中,主要是两(三)种类型的容器: 顺序容器(序列容器) + 关联容器  + 无序关联容器(C++11) [即哈希容器]
另外还存在容器适配器(Container Adapter) 以及 span(C++20)

因为我们使用容器, 要容纳各种类型. 所以容器都是模板实现, 故容器是STL的重要组成部分之一

我们就来看看, 容器的使用究竟都需要注意些什么方面吧!

## Item 01: Choose you containers with care

是的, 可供我们选择的容器中类相当多. 如何选择适合我们的容器便成了一个问题.
我们先从各种容器的特点来进行分析吧, 这样采访便我们确定适合自己使用的类型.

发展至今, 完整的容器列表如下:

|顺序容器|注解|
|------|
|vector|向量,说的简单点就是变长数组(非VLA), 支持尾部的插入, 弹出.适合构造栈|
|deque|双端队列, 支持从首尾进行元素的插入和删除|
|list|双向链表, 支持首尾的元素插入和删除|
|array (C++11)|将内置数组STL化, 支持STL的通用操作, 实用性一般.[不过为了STL规范化]|
|forward_list (C++11)|单向链表, 手写的性能最好的单向链表|

|关联容器|无序关联容器|注解|
|-------|-----------|
|map|unordered_map (C++11)|K-V对集合,底层使用自平衡二叉查找树|
|set|unordered_set (C++11)|唯一键集合|
|multimap|unordered_multimap (C++11)|(允许重复)K-V对集合|
|multiset|unordered_multiset (C++11)|(允许重复)唯一键集合|

|适配器|注解|
|-----|
|stack|堆栈适配器, 一般使用vector, deque构造|
|queue|队列适配器, 一般使用deque构造|
|priority_queue|优先队列|

|相接容器|注解|
|--|
|span (C++20)|...,看Reference吧,C++20为此还添加了相接迭代器|

关于迭代器非法化:

**只读方法决不非法化迭代器或引用。修改容器内容的方法可能非法化迭代器和/或引用，
**

按照常理我们进行容器的选择时, 按照这样的思路: *容器的特点来进行考虑*

例如:
- vector擅长在尾后添加/删除元素
- deque擅长在首尾添加/删除元素
- ...

然而我们现在要考虑的不仅仅是这些, 我们可以从下面这些方面入手:

1. 是否需要在任意位置插入元素
2. 是否关心元素在容器中的排序情况 (此处强推哈希容器, 即无序关联容器)
3. 需要何种类型的迭代器 (vector中是任意访问迭代器, List中是双向迭代器)
4. 查找速度是否是关键因素 (无序关联容器 > 排序vector > 序列容器)
5. ...

以上只是我们提出的一些选择容器的建议, 本质上还是要依靠: **我们对于不同容器的熟悉程度决定**

## Item 02: Beware the illusion of container-independent code

STL的设计概念是泛化的思想.没错,它在想办法将元素存放在容器中实现为类型无关的 (使用Template)

也许就会引发出一个问题: *你想尝试编写与容器类型无关的代码, 以此来实现更高程度的泛化*

可行吗 ? **一定不可行**

可以稍微考虑一下: 序列容器提供了`Container::push_back`,`Container::push_front`等方法, 关联容器则提供了`Container::lower_bound`, `Container::upper_bound`, `Container::equal_range`等方法

这种你如何编写容器无关的代码 ? ? ?

那么,范围缩小一点. 对于同时序列容器, 我们比较`list`和`vector`

`list::splice`完成列表的链接工作, 保证常数时间.
`vector::reserve`用于消除重分配,

这两个小的能力,在`list`与`vector`之间可以实现互操作吗？

**答案否定的！**

**最后, 如果我们能够实现真正"泛化"的容器代码, 其所能提供的功能一定是极为有限的**
**这种东西, 还能够叫做STL吗 ? **

究其原因, 实质是因为：　**不同的容器是为了不同的功能而设计, 而且他们也提供了不同性能的迭代器**
因此我们尝试编写容器无关的代码, **从出发点就是错误的, 不同容器有不同用途, 怎么能轻易泛化呢?**

但是, 如果真的有一天, 我们需要进行容器的修改, 我们该怎么办呢?

一行一行代码去修改吗? 那肯定是不现实的.

我们可以提供封装技术. 最基础的封装就是类型定义`typedef/using`

```cpp
class Weight{};
using WidgetContainer = vector<Widget>;
using WCIterator = WidgetContainer::iterator;

WidgetContainer cw;
widget temp;
auto it = find(cw.cbegin(), cw.cend(), temp);
```
在我们有需要的时候进行替换即可.

再高级一点的封装便是使用面向对象的class机制了

## Item 03:  Make copying cheap and correct for objects in containers.

曾经在EffectiveC++的系列博客中, 我们提到过: 我们推荐pass-by-reference 的形式进行值传递
我们也说过, C++是一个语言联邦, 我需要根据目前所处的不同领域来决定我们应该使用的方式.

**在STL中, 使用的是pass-by-value的形式**

没错, 我们将值拷贝进入容器中, 进行操作, 最后又将值拷贝出来.

整个STL的工作方式就是进行拷贝, 为什么呢? 
*Container是提供容器功能的, 我们可以同时将一组数据装在不同的容器中进行不同需求的操作*
*这样, 你明白了, 我们为什么处理拷贝了吗 ?*

那么,它如何实现呢? 是的依靠你的`copy constructor` / `copy assign operator`

那么, 比如我们提供下面这样的类型,

```cpp
class Temp {
public:
    Temp();
    Temp(const Temp &) = delete;
    Temp operator=(const Temp &) = delete;
    Temp(const Temp &&) = delete;
    Temp operator=(const Temp &&) = delete;
    ~Temp();
};
```
它是无法使用容器的.

```cpp
{ ::new(static_cast<void*>(__p)) _T1(std::forward<_Args>(__args)...); }
       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
// 其中 _T1是构造函数, __args是模板变参. 这里的构造, 委托给对象的拷贝构造函数
```

即使使用C++11,新增加的`Container::emplace_back`,直接构造,也需要使用`Temp::copy_constructor`

那么, 我们实际进行容器使用的时候, 相当需要注意的一个问题便是: **splice down(剥离)**

指的是: **对于基类实例化的容器, 我们存放派生类的对象, 就会导致派生类对象中的派生部分剥离**

=> **会导致, 我们使用多态机制失败, 至于为什么, 不言自喻**

那么, 我们有什么办法呢? **指针**, Emmm, 是个不错的想法, 但是指针太危险了

有什么办法呢? 我们之前在EffectiveC++的内容中提到过, 智能指针. 是的,这是个不错的想法.

## Item 04:  Call empty instead of checking size() against zero.

使用`Container::empty()` 替换 `Container::size()`

这是问题吗? 是的, 在追求效率的STL面前, 适当的编码是真的可以提高效率

为什么呢? 这个问题
**原因就是: STL规定了实现要求的时间复杂度, 却不限制实现方式**

而`.empty()`是常数时间, `.size()`对于某些容器并非常数时间.

我们来看这样的例子:
```cpp
std::list<T>::empty();     // 空
std::list<T>::size();      // 大小
std::list<T>::splice();    // 链接操作
```

对于标准容器`list`, 我们将它作为链表来使用, 也即是说: 我们要求它擅长进行节点的创建与链接操作

因此, `list::splice`的实现被要求是常数时间, 即链接节点的时间复杂度与链表长度无关.
所以, 这也就意味着, 我们在`list::size`实现时, 就达到了线性时间, 因为它必须遍历链表才能求出长度
所以, 你懂了吧,为什么我们使用`empty()`而非`size() == 0`进行空容器的判断

*当然, 只是针对于splice是常数时间, size是线性时间实现的STL, 如果有其他实现, 我们便要阅读文档判断*

STL是追求高性能的标准库设施, 高效的使用STL, 也是我们所努力的目标.

## Item 05: Prefer range member functions to their single-element counterparts.

如果说之前的手法都是小打小闹, 那么这一点, 则是明显提高效率的手法 !

Scott Meyers先生首先提出了一个**将某容器c后半部分如何拷贝进vector**的问题.
正确答案是 `vec.assign(c.cbegin() + c.size() / 2, c.cend());`

很简洁明了, 不是吗? 另外我要强调的是: **这同时还是相当高效的操作**

首先是: 我们经常会写成循环的方式, (~~如果你写成多重循环, 拖出去斩了~~)

那么, 这里便是我们所要进行阐述的地方: **使用区间形式的调用 替换(replcae) 单元素形式的调用**
是的没错, 正是这样: **区间形式的操作, 比你想像中效率要好一大截**

主要基于以下三大原因: [这部分主要针对顺序容器]
1. 区间形式的调用, 减少多次函数调用, (当然`inline`就不会有额外开销,但这个很难是`inline`实现)

2. Emmm ,你仔细想想.顺序容器我们之前说过, 它也是连续内存的容器, 
所以: **进行元素的修改, 会导致元素的挪移,没错吧 ? **
那么, 我们如果使用区间形式的调用 => **是的, 一次到位, 减少了相当大次数的挪移(具体数据自算)**

3. 第三个, 是针对STL的动态扩展特性来说的, 因为其动态的特性, 所以我们才舍弃内置数组/C-String
当然, `std::array(C++11)`可以了解一下(统一规范, 没有什么大用...)
来了, 关键的: **插入, 修改此类操作, 经常超出容器的容量(区分容量/大小), 会导致重分配**
重分配一般涉及 **重分配 -> 拷贝构造 -> 析构 -> 释放内存**
按照, 大多数`vector`的实现规范, 插入1000个元素, 基本上近似于10次重分配
而如果我们使用**区间形式调用, 还是同第二点, 一步到~~胃~~, 不对, 一步到位, 效率增长可想而知**

听了上面的三点分析, 是不是恍然大悟呢? 我们还没说完:

> Tips:
> 1. 我们最开始展示了`Container::assign`的用法, 旨在说明, 还存在方便的成员函数可用
>  但是, 我们更范用性的操作是`Container::insert`, 赋值也是一种插入嘛.
>  不过, 我个人(当然是Scott先生提到过), `std::copy`其实有相当大的误导性
>  STL, 就是拷贝, 使用`copy`一方面是会误导我们针对明确语义的`assign`, `insert`操作
>  更重要的是, 一般实现, copy展开之后, 基本就是显式循环, **是的, 上面三个分析,不复存在**
>  即使你的确使用了区间形式的调用...

> 2. 我们针对第二点, 需要注意的是, 我们能够一步到位的运算挪移, 
> 是要建立在迭代器支持的基础上的, 这种迭代器, **前向迭代器往上**
> 但是, 一般容器提供的都达到了这一步, 
> 除非是**输入迭代器(等同于单元素调用, 步步移动)**, 不然不需要考虑这一点.

*虽然有上面两个tips, 但是, 好像并无大碍, 不是吗 ? *

那么,我们需要注意的是什么? **该关联容器了**

**众所周知, 关联容器, 一般是基于节点的容器, 实现上经常是链表/树等**

*那么, 我们的分析还实用吗 ? *

我们对于关联容器要从这一方面来切入: 就拿list来说.
![std::list::insert](http://www.qiniu.evilcrow.site/EffSTL_list.png)

其中要干什么? **修改指针指向, 没错吧?**

**那么,单次循环插入, 和区间一次链接, 做进行的操作开销, 懂了吧?**

*是的, 指针赋值开销很低, 但是, 不付出开销岂不是更好 ? *

那么, 我们总结一下:

1. 插入(insert)  建议使用`std::COntainer::insert(pos, c.cbegin(), c.cend())`
2. 赋值(assign) 建议使用`std::Container::assign(c.cbegin(), c.cend())`
3. 删除(erase) 建议使用`std::Container::erase(c.begin(). c.end());`

是的, 除非你有足够的理由能够推翻三驾马车, 否则,使用区间形式的调用是更好的选择

## Item 06:  Be alert for C++'s most vexing parse

摆脱C++烦人的分析(parse)机制, 是的,这个问题主要是进行函数原型的诊断所导致的问题

函数原型会有什么问题 ?
```cpp
class Weight{};

Weight w();            // Error !
Weight w;              // 使用default constructor
```

是的, 我们在C++中调用默认构造函数, **一定不能加上括号, 否则会被parse => function prototype**

这种创建对象还比较明显, 下面这种例子就相当隐晦了

```cpp
std::for_each(istream_iterator(file), istream_iterator());
```
看上去并没有什么问题, 但这也只是看上去没有什么问题..
这其实是一个函数原型的声明

我们来分析一下: 
1: 类型为`istream_iterator`名为`file`的变量.
2: 返回值为`istream_iterator`类型, 无参数的函数指针 [此处省略函数名, 所以可以没有(*fp)]

看看我们下面的正常用法:

```cpp
void foo(int (a), void(int));             // 我们在函数原型中是可以省略参数名的

int main(void)
{
    void func(int);                       // UNP中看到的邪教, 其实声明也不一定放外面
    foo(2, func);
}

void func(int a) {...}

// 同时, 此处说明函数与函数指针, 用法相同, 不过语义是不同的. (指针 | 函数)
void foo(int a, void fp(int)) {...}       // 定义中, 因为我们要使用参数, 所以函数名不可省略
```

**我们在此处着重强调的是: 不要因为分析机制而产生误用!**

## Item 07: When using containers of newed pointers, remember to delete the pointers before the container is destroyed.

这一条款, 我觉得可以是 EffectiveC++中 Chapter III 资源管理的扩展 (具象化)

**当容器中使用new的到的指针时, 在容器销毁时, 一定要delete掉**

这是怎么说? **因为C++中析构和释放内存是两码事**

我们从容器中退出时, 它仅仅是, 对象销毁(是的, 指针销毁掉了), **但是, 相关的内存并没有释放掉**
**因此造成了内存泄漏 !(Memory Leak)**

来看个例子吧:

```cpp
void func()
{
    std::vector<Weight *> vec;

    for (...)
        vec.push_back(new Weight); // vec.emplace_back(new Weight)
    ...
    ...
}                 // Memory Leak
```

那么, 单纯的构造却不delete势必会造成问题.

如果这样呢?

```cpp
void func()
{
    std::vector<Weight *> vec;

    for (...)
        vec.push_back(new Weight); // vec.emplace_back(new Weight)
    ...
    for (auto &var : vec)          // Iterator 遍历当然也行
        delete var;
}                 // Memory Leak
```

这种做法情况比上面好一点, 也只是好一点....
**因为, 要是第一个delete的时候, throw exception了, 后面又都是内存泄漏**

那么, 还有办法吗? 有的!  [其实是个换皮怪]

```cpp
// 因为我们会修改容器中的元素, 所以不使用Container::c[begin | end]();
for_each(vec.begin(), vec.end(), UnaryFunction);
```

那么, 这个UnaryFunction怎么写 ?

STL中有这么些概念: 谓词, 基类, 函数适配器. (自行了解, 或者我后面会提及)

```cpp
// UnaryFunction的处理方法

// 1. 使用function object (其实就是函数重载运算符)
template <typename T>
class DeleteObject : public unary_function<const T*, void> {
public:
    void operator()(const T *pointer) {
        delete pointer;
	}
};

for_each(vec.beign(), vec.end(), DeleteObject<Weight>());

// 2. lambda算式

// C++14 支持lambda中auto推导, C++11, lambda必须写成具体的类型, 使用template又略显粗糙
for_each(vec.begin(), vec.end(), [](auto &var){ delete vec; }); 

// 3. 使用bind() C++11 提供, 不同于之前的 bin1st, bin2nd
// 语法丑陋, 暂不展示 [主要是bind提供适配功能, 还要依靠其他函数...]
```

**此处, 因为历史原因, 但是现在lambda一个式子就可以把函数对象, 适配器, bind通通吃掉**

如果你很喜欢写函数对象也行 (lambda到std::function是不会隐式推导的, 这个时候使用函数对象适配)

关于上面的函数对象例子, 我们必须是知道: **容器元素为Weight, 才能操作, 那么又其他办法吗?**

有!

```cpp
class DeleteObject {
template <typename T>
public:
    void operator()(const T *pointer) {
	    delete pointer;
    };
}

for_each(vec.begin(), vec.end(), DeleteObject());    // 会自行根据传入的参数推断类型
```

有什么好处吗? 有的!
*比如有的类没有虚析构函数, 那么,你继承之后, 如果误以为他有虚析构函数, 而使用基类指针delete*
懂了吧?

但是我们使用模板的形式, 保证自动类型推导. 
**是不是写的代码少, 还反而准确呢? [是的, 代码越少, 错误越少, 乱***装13就是作死]**

但是, 回到我们的主题上来:  `std::for_each`只是省得写显示的循环, 还是会有内存泄漏的风险

*即使是C++20中的提供policy的形式, 也只是可能能够不按顺序操作, 但这一切都是徒劳*
*可能只有std::TS中的并行算法, 还有机会解决这种问题*

真的无解了吗? 怎么可能?

想想我们之前在EffectiveC++中的解法: **是的, 使用类管理资源, RAII !, 具体下来就是 智能指针**

```cpp
void func()
{
    std::vector<std::shared_ptr<Weight>> vec;
    
    for (...)
        vec.push_back(std::shared_ptr<Weight>(new Weight));
    ...
    ...
}       // Memory must be released
```

使用智能指针的形式, 可以保证无论是手工忘记, 还是异常抛出, 资源都一定不会泄漏!

## Item 08:  Never create containers of auto_ptrs.

这一条款是在说: `std::auto_ptr`类似于`std::unique_ptr`, 在使用中会被置空, 交出资源.

auto_ptr并非真正意义上的智能指针, 它是历史上一个实现不完全的, `std::unqiue_str`

鉴于至今, 我们已经有`std::shared_ptr`和`std::unique_str`以及避免环回的`std::weak_ptr`

所以, 此条款, 我们不再讨论.

## Item 09:  Choose carefully among erasing options

之前我们重点在如何往容器中添加元素, 现在我们来聊聊删除元素的手法
对于如何删除容器中的元素的手法, 我们分为三类,三个层次说明:

### 删除元素

1. 顺序容器, 我们使用erase-remove的手法

 `Container::erase`是它擦除未指定值并减小容器的物理大小,
`std::remove`是迁移（以移动赋值的方式[C++11]）范围中的元素进行移除。保持剩余元素的相对顺序，且不更改容器的物理大小 [即是说, 从remove返回到end()的迭代器不失效, 仍可使用]

 所以, 我们将这两个组件配合起来. 使用remove-erase的模式

 *为什么要使用`std::remove`直接`Container::erase`不是也能达成效果么? *
**想想之前说的, 区间形式的调用和遍历容器单元素调用, 懂了吧 ?**

 基本形式是这样:
`vec.erase(remove(vec.begin(), vec.end(), val), vec.end());`

2. `std::list` (是的, list虽然是顺序容器, 但是它的实现方式又类似于关联容器, 所以要单独拿出来说)

 怎么玩? 直接`list.remove(val)`即可,,为什么不区间 (因为`std::list::remove`之间移除指定val的元素)

3. 关联容器,我们直接erase即可, 同时保证是对数时间开销
 **注意: 标准关联容器, 没有remove成员函数, 且使用算法可能覆盖容器的值, 更甚至于会破坏容器**

### 按照判别式来删除

也就是说, 某个满足条件的值被删除, 不仅仅是某个指定的值了.

1. 顺序容器, 将`std::remove`更换为使用`std::remove_if`一切同往常使用

2. `std::list`我们同样使用`std::remove_if`即可

3. 关联容器, 这怎么办呢?
 
 不不不, 不能这样来, 因为,关联容器的`Container::erase`使用之后会是的指向此元素的迭代器失效
 什么意思, 即我们在循环中, 使用`++`的时候, 已经是是在一个无效的迭代器上操作, 结果是未定义的
 那么,怎么办? **保存迭代器的值, 然后进行操作即可, 好办法**
 
 *另一种办法是, 使用Copy_and_Swap的手法, 通过`std::remove_copy_if` + `std::swap`操作*
 ```cpp
 for (auto i = c.begin(); i != c.end(); ++i) {
     if (BadValue(*i))
         i = c.erase(i);                  // 操作成功 !
 }
 ```
 
 上面的操作是难能可贵的, **因为在C++11之前,关联容器的**`Container::erase()`**返回void**
 也即是说: **我们无法得到删除操作后, 下一个合理的位置!**
 之前的做法是 `c.erase(i++);` 籍此保证迭代器未失效
 所以说: C++11真的是带来了巨变, 为我们提供了高效的操作

### 删除元素和额外的操作

怎么说呢?我们这一类说的是, 实际中经常使用的情况: 比如志记需求 (~~完了, 迷上侯捷的说法了~~)

1. 顺序容器
 
 **呦霍, 完蛋咯 ~**
 现在无法使用算法了, 因为我们无法在其中进行额外操作. (需求肯定是优先效率的)
 怎么办呢? 而且 **顺序容器删除后, 后面所有元素的迭代器都会失效的, 因为它基于连续内存分配**
 
 想想上面关联容器的做法:
 ```cpp
 for (auto i = vec.begin(); i != vec.end(); ++i) {
     if (BadValue(*i)) {
	     log(..);
         i = vec.erase(i);
     }
     ...
 }       // 同理即可 , OK!
 ```

2. `std::list`
 
 做法同其他顺序容器
 
3. 关联容器
 
 同我们之前的讨论, 是不是很清晰了呢 ? 

**小小总结一下:**

对于直接删除: `remove-erase` + `std::list::remove` + `Loop: Container::erase`
条件删除: `remove_if-erase + lambda` + `std::list::remove_if` + `Loop: Container::erase`
额外操作: `Loop: return value` + `std::list::erase` + `Loop: return value`

## Item 10:  Be aware of allocator conventions and restrictions

Allocator是一个相当重要的内容, 但是也没有想想中那么重要... (历史遗留问题比较严重)

因为这部分内容是真的比较操蛋, 所以我拣重要的来说, 至于深刻理解分配机制: 等我STL源码吧.

重要的内容有下面几点:
1. 分配器是一个模板, 其实不难理解, 因为他要针对各种STL容器来进行施用
2. 提供类型,`Allocator::pointer`与`Allocator::reference`, 但是, 始终为`T *`与`T &`
3. 尽量不要使Allocator含有状态, 也就是说: **避免分配器中使用非静态成员**
[因为我们对于同一容器特化的不同实例, 都要能施用相同的分配和释放操作]
4. `std::Allocator`的使用习惯与`new operator`并不相同, 体现在函数参数传递与返回值上
5. `std::rebind`是施行于某些类型上的关键 !

我们在这里详细来说`std::Allocator::rebind`, 它其实只是一个提供类型的, --- **模板**

举个例子: `std::list`的普遍实现采用了链表的形式, 也就是说, 一般是这样:
```cpp
template <typename T, typename Allocator = allocator<T> >
class _list {
class ListNode {
public:
    ListNode(T x) : val(x), left(nullptr), right(nullptr) {}
    int val;
    ListNode *left;
    ListNode *right;
};
...
};

// list内部使用
class _list {                            // class不像namespace, 它必须是连续的
...
std::allocator<T> allocator_list;     // 使用
...
}
```

是的, 我们`list`的类型是`int`, 但是我们要分配的是`int`吗? **否 !**,我们要分配的是: **ListNode**

于是, `std::allocator::rebind`的作用出来了: **重新绑定元素类型, 以供分配器使用**

一般是这样的形式:

```cpp
template <typename T>
class allocator {
public:
    template <typename U>
    struct rebind {
        typedef allocator<U> other;
    };
    ...
};

list<T> => Allocator = allocator<T> => 
typename Allocator::rebind<ListNode>::other =>
allocator<ListNode> [即为所需分配器] [rebind这个叫法也挺形象的]
```

## Item 11:  Understand the legitimate uses of custom allocators

紧接着上一条: 这条是自定义分配器的用法

对于这部分内容: 粗略地说两句

首先, 符合标准Allocator一样, 提供pointer, 以及reference.

并且不保存状态(即只有静态成员), 而且, 符合Allocator的使用习惯.

这些内容, 反正说的比较含糊,... 实用性一般, 具体用到再细说吧.

## Item 12: Have realistic expectations about the thread safety of STL containers.

对于STL, 我们不能对其线程安全性有过多的期望, 何出此言?

标准只是期望: 多线程读容器OK, 多线程写容器OK.

但是这些只是期望而已, 你不能对其有依赖, 因为有的实现符合, 有的实现并不支持.

那么, 我们要求STL提供线程安全会怎么样? 

**没有好下场.**

将线程同步的内容置于STL的实现中, 想法很美好,但是这无疑会让STL变得异常冗余复杂
**使得STL丧失了其高效快捷的特点, 而且,退一步讲, 万一不需要线程支持, 那么多余的工作反而影响效率**

基于上面的分析: 我们不要求STL来提供线程安全的支持, 我们使用手动线程同步

怎么办呢? `std::mutex` + `std::cond_varible` 一般使用互斥锁 + 条件变量就OK

但是, 死锁的问题, 还在威胁着我们.

想想看, lock也不就是一种资源么? 对于资源, 我们一般怎么办? Resource Management

**没错, RAII !**

标准库中, `std::shared_ptr` 和 `std::lock_guard` 是已经实现的RAII实例

通过使用这些设施 ,以及我们手动的RAII, 便可以自行完成线程安全的STL使用

下面是C++Reference中, 对线程安全的描述: [其中标识了一部分保证的线程安全操作]

![线程安全](http://www.qiniu.evilcrow.site/EffSTL_thread_safety.png)

***

STL的内容真的好多, 实现想必是相当的精彩, (xxx之前一直喷,但是我觉得还可以, 就是写的风格不太好)

最近确实火烧眉毛了, 但是EffectiveSTL还不错, 挺有意思的.

其实我现在最困惑的时: 如何能够详实的将自己学到的技术落实下来.

**可能只能多看, 同时写的时候,一开始刻意去用吧, 不然真的是有点尬, , , **

怎么说呢, 迅速结束这些, Network Programm的内容 + Operation System的内容也不会少,

管他呢, 干就完事了.

November 10, 2018 5:35 PM
