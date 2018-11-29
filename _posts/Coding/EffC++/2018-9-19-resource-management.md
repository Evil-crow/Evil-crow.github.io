---
layout: post
title: <星月夜> Resource Management
date: September 19, 2018 7:36 PM
excerpt: Chapter III 资源管理
categories:
- C/C++
tags:
- EffectiveC++
toc: true
comments: true
---

*C++是一门强大的编程语言, 它自信程序员拥有强大的本领驾驭它的各个方面, 所以将资源管理的任务*
*全部托付给程序员(好吧, 只是为没有GC在洗地...)*
*从最基础的动态内存分配 ~ 文件描述符(file descriptor) ~ 套接字(socket) ~ 线程(thread)*
*资源管理的任务无处不在, 因此资源管理便成为我们严格对待的方面*

## Item 13: Use objects to manage resource

对于资源管理, 说的简单了, 其实最重要的就是: **资源泄漏(memory leak)的问题**
1.不使用的资源, 应该及时释放
2.已经释放的资源, 绝对不能使用, 会导致未定义行为

我们从最常见的heap-based资源开始, 先来看看我们平时是怎么做的吧,
```cpp
int *pi = new int;                 // allocate resource(int) on heap

int *get_resource(...);	        // func -- get resource

void func()
{
    int *p = get_resource();      //request resource
	...
	delete p;                     // release resource
}
```
正如我们上面的编码, 我们先申请资源, 最后释放资源, 一切看上去是那么的井然有序.
**真的能如你所愿吗 ? **
*考虑一下: 如果在`...`中间,存在: 提前return, throw异常, 多重嵌套, 还会正常工作吗 ?*
**事实上, 只要是足够正式的工程代码, 极有可能工作流是到不了 资源释放的步骤的.**

当然, 通过细致而严谨的编码, 我们可以做到资源管理的安全性
但是其中的难度可想而知
另一方面, 如果并非是代码维护者, 对于代码做了某些改动, 极有可能导致资源管理的有一次失控

**因此, 引入使用对象管理资源的思想, 即资源获得即初始化(RAII)**

> 那么, 何为RAII?
>
> RAII(Resource Acquisition Is Initialization)
> 字面意思为: 资源获取即初始化. 是有点拗口
> 实际上,它遵循了两个思想:
> 1. 资源一旦初始化(/赋值)即交给对象管理
> 2. 资源的释放, 交由类机制的析构函数释放

那么, RAII的优点体现在哪里?
1.将资源交给对象管理, 只要处理的得当了, 可以避免内存泄漏(某种程度上)
2.隔离了资源的所有权, 解放了程序员, 就是说: 简化了资源管理的任务

RAII的资源管理策略, 是真的十分巧妙 !

那么, 如何使用RAII ? 
分为两类: 类库中已经采用RAII策略的设施以及手动构建RAII类(Resource-managing classes)

### Standard Library Facilities

目前, 在标准库中已经采用RAII的设施有: 
`std::shared_ptr`, `std::unique_ptr`, `std::lock_guard`
**我们建议对于heap-based资源, 使用这些设施进行管理**

例如:
```cpp
void func()
{
    /*
	 * int *p = new int;
	 * std::shared_ptr<int> p1(p);
	 */
    std::shared_ptr<int> p1(new int);     // acquire resource when request
}                                         // leave the block with releasing resource
```

```cpp
void func()
{
    std::lock_guard<std::mutex> lock(&mtx);    // std facility, not heap-based resource
}
```

这里面我们使用了, `std::shared_ptr`, `std::unique_ptr`. 这是现在C++ (C++11起)标准库设施.
在< EffectiveC++ >, 一书中提及的, `std::auto_ptr`(已废弃, 算是`std::unique_ptr`早期版本)
`std::tr1::shared_ptr`已经正式纳入标准库, 并非是TR1, TR2扩展, 且TR1, TR2扩展已经废弃

不过,使用智能指针好也罢, 不使用也罢.
我们需要注意的是: 
**智能指针的析构, 使用delete, 非delete[]**
**也即是说, 我们对于数组的动态内存分配, 要不然拒绝, 要不然自行构建删除器**
**毕竟, 对于数组类型, 标准库提供了自动析构的, vector以及string等设施(顺序容器)**

### Resource-managing classes

对于常见的其他资源, 比如:
`socket`, `file desdcriptor(fd)`, `thread`, `mutex`等等, **我们更倾向于手动构建RAII类**

例如:
```cpp
class lock {
public:
    lock(std::mutex &mutex);
	~lock();
private:
    std::mutex mtx;
};

Lock::Lock(std::mutex &mutex) : mtx(mutex) { std::lock(mtx); }

Lock::~Lock()
{
    std::unlock(mtx);
}

// Usage
void func()
{
    std::mutex a;
	Lock lk(a);
	...
}                  // release resource automaticlly
```

```cpp
class Socket {
public:
    Socket(int fd) : socket_fd(fd) { ; }
	~Socket() {
	    ::close(socket_fd);
	}
private:
    int socket_fd;
}

// Usage
void func()
{
    int fd = ::socket(...);
    Socket s(fd);
	...
}                // release resource automaticlly
```

> 请记住:
>
> 1. 为防止资源泄漏. 请使用RAII对象, 它们在构造函数中获得资源并在析构函数中释放资源
> 2. ~~两个常被使用的RAII Classes 分别是 `std::tr1::shared_ptr` 和 `std::auto_ptr`~~
>  ~~前者通常是较佳选择, 因为其Copy行为比较直观.若选择`std::auto_ptr`,~~
>  ~~复制行为会使它指向null~~

## Item 14: Think carefully about copying behavior in resource-managing classes

Emmmmmm, 严格意义上来讲, RAII资源类, 一般而言, 是禁止拷贝的.
对, 没错.
但是, 什么事情都有例外, 万一真的出现需要拷贝的情况, 我们应该作出什么样的选择呢?

常见的选择有下面四种:

### No Copy

这是最常选择的策略, 因为RAII类没有拷贝的必要与需求, 我们何必自找麻烦呢?

那么, 禁止拷贝的手段呢? (之前的条款已经提到过了)

两种手段: 
1.使用Uncopyable/noncopyable 
2.明确语义, 不需要拷贝的函数

```cpp
// Boost::noncopyable

class noncopyable {
protected:
#if !defined(BOOST_NO_CXX11_DEFAULTED_FUNCTIONS) && !defined(BOOST_NO_CXX11_NON_PUBLIC_DEFAULTED_FUNCTIONS)
    BOOST_CONSTEXPR noncopyable() = default;
    ~noncopyable() = default;
#else
    noncopyable() {}
    ~noncopyable() {}
#endif
#if !defined(BOOST_NO_DELETE_FUNCTIONS)
    noncopyable(const noncopyable &) = delete;
    noncopyable &operator=(const noncopyable &) = delete;
#else
private:
    noncopyable(const noncopyable &);
    noncopyable &operator=(const noncoptable &);
#endif
};
```

```cpp
// using Boost::noncopyable

class Lock : public boost::noncopyable {
public:
    Lock();
    ~Lock();
private:
    std::mutex mtx;
};
```

```cpp
// delete copy function
class Lock {
public:
    Lock();
    ~Lock();
    Lock(const Lock &) = delete;
    Lock &operator=(const Lock &) = delete;
private:
    std::mutex mtx;
};
```

### Share Resourse

除了禁止拷贝之外, 我们一般还可以接受, 共享资源.
即指: 不同的pointer/reference可以指向相同的资源.

我们一般有这两种手段: 使用智能指针RCSP, 或模拟底层引用计数
这两种自然就是 heap-based资源 与 其他类型的资源
```cpp
// using std::shared_ptr (not std::tr1::shared_ptr)

class Lock {
public:
    Lock();
    ~Lock();
    Lock(const Lock &lk) {                   // 实现太shi, 差不多这个意思
        this->pmtx = lk.handle();
	}
    std::shared_ptr<std::mutex> handle() {
        return pmtx;
	}
private:
    std::shared_ptr<std::mutex, std::unlock> pmtx;        // 将std::unlock作为删除器
};
```

```cpp
// Using reference-count
class Lock {
public:
    Lock();
    ~Lock();
    Lock(const Lock &lk) {                   // 实现太shi, 差不多这个意思
        mtx = lk.handle();
		refCount++;
	}
	std::mutex &handle() {
        return mtx;
	}
private:
    std::mutex mtx;
	long int refCount;                       // 引用计数
};
// 细节没注意, 基本上就是这个意思
```

### Move Reference

此部分, 和上一部分类似. 其中的区别便是: 转移资源而非共享资源

*使用现代C++, 我们很容易想到标准库设施*
`std::unique_ptr`进行使用, 其他类的资源(非heap-base)自然就是`std::move()`了

### Deep Copy

这种需求,真的是比较少, 本身RAII资源进行拷贝就少, 深拷贝更是凤毛麟角, 这部分就不再赘述
记住: 深拷贝, 拷贝的不仅是指针/引用等, 同时还拷贝了其所指向的对象. (有什么用, 反正现在没用上)

上述我们介绍了四种策略, 但是从真正意义上来讲:
**我们仅仅接受: 禁止拷贝 + 共享资源**
*转移资源/深拷贝 并非是常见且合理的做法*

> 请记住:
>
> 1. 复制RAII对象必须一并复制它所管理的资源, 所以资源的Copying行为决定RAII对象的Copying行为
> 2. 普遍而常见的RAII class copying行为是: 抑制copying, 施行引用计数法. 不过其他行为也都可能被实现

## Item 15: Provide access to raw resources in resource-managing classes

*本条款的标题叫做: 在RAII资源类中提供访问资源权限*

**说实话, 看着矛盾是吧? 我们好不容易封装个资源, 现在又要开放接口, 搞事情 ? **

**客观分析: 其实这并不矛盾, 为什么? 因为我们使用class管理资源的目的并非是为了封装 ! **
**是的, 你没听错, 我们使用class管理资源的主要目的是为了能合理的, 管理和释放 ! **

**我们甚至可以, 谨慎的实现成员函数来避免直接访问资源, 但是与APIs的交互又该怎么办 ? **

*并且, 纵观标准库中的RAII设施, 同样提供了接口去访问资源.*
比如: `std::shared_ptr::get()`等

**所以, 我们既然本身目的并非是为了封装, 所以我们开放访问资源的接口并非不合理**

下面介绍访问资源的两种方式

```cpp
// Using interface get() etc.
class Lock {
public:
    Lock();
    ~Lock();
    std::sahred_ptr<std::mutex> &get() {
        return pmtx;
    }
private:
    std::shared_ptr<std::mutex, std::unlock> pmtx;
};
```

```cpp
// using operator TYPE()
class Lock {
public:
    Lock();
    ~Lock();
    operator std::shared_ptr<std::mutex>() const {
        return pmtx;
	}
private:
    std::shared_ptr<std::mutex, std::unlock> pmtx;
};
```
```cpp
// Usage

void foo(std::shared_ptr<std::mutex> &mtx)
void func(lock lk)
{
    foo(lk.get());    // get()
	foo(lk);          // operator TYPE()
}
```

那么, 我们如何抉择使用哪种方式呢?

**各有优劣**
1.使用显示接口, 可以是我们的意图更加明显, 除了某些极端人士嫌丑陋以外, 也没什么不好的.
2.使用隐式转换, 其实容易出问题, 有的时候会出现相反的效果

> 请记住:
>
> 1. APIs往往要求访问原始资源(raw resource), 所以每一个RAII class应该提供一个"取得其所管理之资源"的方法
> 2. 对原始资源的访问可能经由显示转换或隐式转换.一般而言显示转换比较安全, 但隐式转换对客户比较方便

## Item 16: Use the same form in corresponding uses of new and delete

使用配对的new和delete 看起来其实是很简单的一个道理.

事实上就是一个很简单的道理.

`new TYPE[] -> delete [] p;`
`new TYPE -> delete p;`

为什么要成对使用, 因为如果不匹配, 就会有这样的问题: (类似于之前讨论指针和数组名混用)

若, 使用delete[ ]不匹配, 就有可能过多释放内存, 甚至于正在使用, 或者无法析构的内存

若. 使用delete不匹配, 就有可能少释放内存, 基本上也是未定义行为.

```cpp
// valagrid的分析
==6539== Mismatched free() / delete / delete []
==6539==    at 0x4C2E1E8: operator delete(void*) (vg_replace_malloc.c:576)
==6539==    by 0x4005BD: main (in /home/Crow/a.out)
==6539==  Address 0x5aeac80 is 0 bytes inside a block of size 40 alloc'd
==6539==    at 0x4C2D8B7: operator new[](unsigned long) (vg_replace_malloc.c:423)
==6539==    by 0x4005A8: main (in /home/Crow/a.out)
```

另外强调两个点:
1.对于数组之类的, 我们更建议vector, deque, string等设施
2.尤其是typedef等使用手动new, delete简直是毁灭级的, 因为它到底用的哪种new我们一无所知

所以, 一定要保证自己**成对匹配的使用new和delete**

~~我曾经在不会C++的时候, 被g卓越因为这个坑过 \白眼~~

> 请记住:
>
> 1. 如果你在new表达式中使用[ ], 必须在相应的delete表达式中也使用[ ].
>  如果你在表达式中不使用[ ], 一定不要在相应的delete表达式中使用[ ].

## Item 17: Store newed objects in smart pointers in standalone statements

这句话, 看上去有点奇妙, 什么意思呢?

*以独立的语句将newed对象置入智能指针*

看个例子:
```cpp
void func(std::shared_ptr<int> sp, void (*foo));     // prototype
void foo()
{
    ...
    throw exception;
    ...
}

func(std::shared_ptr<int>(new int), foo());
```

这其中, 在func()函数调用的时候, 实参和形参匹配时的求值顺序是未定义的

所以, 就有可能出现,下面这样的顺序

```cpp
1. new p;
2. foo();
3. std::sahred_ptr<int> (...);
```

那么, 一旦在第二步骤的时候, "成功"抛出异常, 毁灭级的打击, 这已经造成内存泄露了
**而且, Debug及其困难, 因为复现的难度已经十分之大 !**

**所以, 应该以独立的语句将new的资源置于智能指针中, 不同语句之间的求值顺序值确定的
**

```cpp
std::shared_ptr<int> pw(new int);
func(pw, foo());
```

**上升一下: 不仅仅是这种申请资源, 只要是需要确定的顺序的行为, 我们就一定要分成独立的语句, 有的时候紧凑, 花哨的编码, 反而是毁灭的入口**

> 请记住:
>
> 1. 以独立语句将newed对象存储于(置入)智能指针!
>  如果不这样做, 一旦异常被抛出, 有可能导致难以察觉的资源泄漏.

*Tips: 我们在本节中讨论的都是资源泄漏(Resource Leak), 已经不仅仅是内存泄漏(Memory Leak)*
