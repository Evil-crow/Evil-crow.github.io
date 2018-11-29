---
layout: post
title: <咸鱼书> 运行时数据结构与内存
dtae: June 4, 2018 9:34 PM
excerpt: 其实这部分才是我觉得咸鱼书真正的精华,有许多书对于这部分要不不提,要不囫囵吞枣一笔带过,但是咸鱼书还是进行了比较详细的分析的,那么我们就来具体看一看整个运行时的数据结构中到底有什么?
categories:
- C/C++
tags:
- Expert_C_Programming
toc: true
comments: true
---

## 一, a.out及其传说

*我们大家接触Linux都是从C语言开始,那么每个人第一个可运行程序都是a.out*

*好奇过为什么用这个东西,做缺省名吗? 哈哈,我其实之前也没有仔细考虑过,现在就来聊一聊*

a.out指的是汇编器输出文件,**但实际上,a.out不是链接器的结果吗? ? ?**

哈哈,这是个历史遗留问题,PDP-11汇编器输出结果为a.out,而当年最正统的UNIX机器便是PDP-11,所以

你懂得,又是个历史遗留问题 !

## 二, a.out与C文件内容之间的关系

C文件的内容如何映射到a.out文件中呢? 那部分填充到哪里呢? 来看看下面的图:

![源文件->可执行文件](http://p8uroi1uf.bkt.clouddn.com/src_aout.png)

如何上图所示,首先我们要强调的是:**段(segement)的概念**, 此段非彼段,与Intel内存模型中的段不同

此处的段是文件中的段,大小为64KB, 最小的控制单位是section

**另外,对于可执行文件,有许多种格式,COSS, ELF, PE格式,我们此处以GNU使用的x86_64elf为查看对象**

其中a.out中有这样几个段:

1. a.out的神奇数字, 用来标识a.out文件的,我们可以通过readelf来查看

2. BSS段, 用来存储未初始化的变量,因为未初始化,所以这个段实际上是不占空间的,他只保存了大小

3. 数据段, 用来存储初始化的数据,(其中还有只读区等小区,不细说),而BSS段 + 数据段即为数据区

4. 文本段, 用来储存代码的,也有叫代码段的说法,一般为.text

这些部分是,**从C源文件 -> 可执行文件的部分(a.out)**,一会儿我们要说到的是:

**从可执行文件 -> 内存映像的问题, 别混淆了呦**

我们在此处可以玩玩这几个命令: size, readelf, objdump ,nm下面我一一来演示:

首先来看看源文件

```cpp
/*
 * @filename:    1.c
 * @author:      Crow
 * @date:        06/04/2018 21:08:09
 * @description:
 */

#include <stdio.h>
#include <stdlib.h>
const int a = 5;    // read_only
int b;              // BSS

void foo(void);

int main(void)
{
    int c = 4;       // stack_heap a.out中看不出来(堆栈毕竟是运行时数据结构)
    printf("Hello World! %d\n", c);
    foo();           // .text
    return 0;
}

void foo(void)
{

}
```

![size a.out](http://p8uroi1uf.bkt.clouddn.com/size_aout.png)

![readelf -a a.out](http://p8uroi1uf.bkt.clouddn.com/readelf_aout.png)

![odjdump -d a.out](http://p8uroi1uf.bkt.clouddn.com/objdump_aout.png)

![](http://p8uroi1uf.bkt.clouddn.com/nm_aout.png)
![nm -n --format=sysv a.out](http://p8uroi1uf.bkt.clouddn.com/nm_aout2.png)

**注意,我们的图是从高地址->低地址, 然而我们的试验结果是地址升序排序**

**从中,我们可以证明,a.out中的顺序的确是,文本段(.text),数据段(.data),BSS段(.bss)**

```cpp
/usr/include/asm/a.out.h
#ifndef _ASM_X86_A_OUT_H
#define _ASM_X86_A_OUT_H
                                                                                          
struct exec
{
    unsigned int a_info;    /* Use macros N_MAGIC, etc for access */
    unsigned a_text;    /* length of text, in bytes */
    unsigned a_data;    /* length of data, in bytes */
    unsigned a_bss;     /* length of uninitialized data area for file, in bytes */
    unsigned a_syms;    /* length of symbol table data in file, in bytes */
    unsigned a_entry;   /* start address */
    unsigned a_trsize;  /* length of relocation info for text, in bytes */
    unsigned a_drsize;  /* length of relocation info for data, in bytes */
};
 
#define N_TRSIZE(a) ((a).a_trsize)
#define N_DRSIZE(a) ((a).a_drsize)
#define N_SYMSIZE(a)    ((a).a_syms)
 
#endif /* _ASM_X86_A_OUT_H */
```

## 三, 到底为什么要如此组织a.out文件

**其实很简单,就是为了可执行文件装入内存中方便**, 来看这张图:

![a.out到内存映像](http://p8uroi1uf.bkt.clouddn.com/aout_mem.png)

对于上面这张图: 我们着重说一下这样几点:

**1. 数据段,文本段直接映射**

**2. BSS段,根据它保存的大小进行扩展**

**3. BSS段与数据段实际上会进行合并, 统称数据区,而数据区往往就是一个程序中最大的段了**

**4. 最后我们再添加上为函数调用准备的堆栈区, 其中具体为stack + heap**

**5. 最下面地委映射区域一般是留给程序扩展,防止跑飞的**

**6. 如果有动态链接库(共享库)之类的,就直接忘堆栈上面怼**

## 四, 堆栈段有什么用? 过程活动记录又是什么?

之前其实和很多学长也都聊过,操作系统中提到的堆栈是栈,数据结构中堆是堆,栈是栈,OK?

堆栈是一种FILO的数据结构,它很适合用于进行函数调用,为什么?

**尤其是我们进行递归调用的时候,需要同时维持多组函数(多个实例)的存在,同时还要保证他们一定的顺序**

**这就很符合堆栈的定义,所以堆栈段的存在很有必要于递归调用**

**也就意味着函数递归调用以外的其他作用,并非是堆栈段存在的必要意义,可以通过其他方式实现**

最最核心的作用就是维护了过程活动记录,CSAPP中也提到过这个东西,就是进行函数调用是的必要信息

以及调用结束之后,我们到底应该如何返回?

那么,我们就来看看过程活动记录吧

![过程活动记录](http://p8uroi1uf.bkt.clouddn.com/process.png)

**上面这个其实就是一个函数在堆栈中的内容了**

**1. 局部变量(local varibales) 即为函数内保存的自动变量(auto)缺省属性**

**2. 参数(arguments) 这其实是,形式参数,用于函数调用的**

**3. 静态链接(static link) 这个与C无关,其实是Pascal,Ada提供的特性,即函数的嵌套声明**

**我们可以通过这个静态链接指针来进行上层函数的访问,从而减少函数之间的通信**

**4. 指向先前结构的指针,很明显,就是指向上层函数(非静态链接),是说上一个过程活动记录**

**5. 返回地址(return address) 不多说了,就是函数调用结束后,返回的位置**

**过程活动记录清晰地阐述了堆栈区的实际意义,以及函数调用的实现**

我们的程序在运行的时候,维护一个指针fp,它指向最靠近堆栈顶端的过程活动记录,之后的全靠指针串起来

```cpp
/usr/src/.../arch/x86/include/asm/frame.h

#ifdef CONFIG_FRAME_POINTER                                                                                                              
 
#ifdef __ASSEMBLY__
 
.macro FRAME_BEGIN
    push %_ASM_BP
    _ASM_MOV %_ASM_SP, %_ASM_BP
.endm
   
.macro FRAME_END
    pop %_ASM_BP
.endm
   
#else /* !__ASSEMBLY__ */
   
#define FRAME_BEGIN             \
    "push %" _ASM_BP "\n"           \
    _ASM_MOV "%" _ASM_SP ", %" _ASM_BP "\n"
   
#define FRAME_END "pop %" _ASM_BP "\n"
   
#endif /* __ASSEMBLY__ */
   
#define FRAME_OFFSET __ASM_SEL(4, 8)
   
#else /* !CONFIG_FRAME_POINTER */
   
#define FRAME_BEGIN
#define FRAME_END
#define FRAME_OFFSET 0

#endif /* CONFIG_FRAME_POINTER */
```

**另外哦我们要注意的一点就是: 过程活动记录很有可能不在堆栈中**

有两方面的原因:

1. 可能是编译器优化,对于依赖程度不高的,我们将过程活动记录储存在寄存器中

2. 对于其他的架构,比如Sun的SPARC,就是使用链表进行过程活动记录的串联,并非存在堆栈中

**时刻记住: 虽然我们常用x86_64架构,但是架构不止这么一种,你去看看linux目录下的Arch目录就懂了**

Arch--Architecture 架构

## 五, 两类内存问题

最后,我们在介绍两种C语言有关却经常会出问题的内存问题:

**首先,说说内存分配的问题,我们可以使用malloc(),free()以及brk(), sbrk()来处理**

**具体的内容,大家下来自行进行了解**

### 1. 内存泄漏

再没有GC(垃圾回收机制)的语言实现中,内存泄漏是个永恒的话题,C中需要谨慎的进行处理

在C++中,有RAII的支持,但是内存泄漏还是个不容忽视的问题

主要是两类原因:

**释放或者改写正在使用的内存**

**未释放不再使用的内存**

我们唯一能做的就是: 慎之又慎的进行内存管理,避免内存泄漏

**另一方面,对于局部对象,我们可以使用alloca()来进行内存分配,这样可以避免退出函数后没有释放内存**

**但是, 局限性很大,对,出了这个函数就凉凉,所以还是好好谨慎的玩吧**

那么,既然它这么危险,我们有没有可以检测内存泄漏的方法?

**有,下米诺安就简单地介绍几个工具: swap(好像只有Sun能用), free, vmstat来查看**

这是最基础的方法,而我们着重介绍的就是: 

Valgrind + kcachegrind

我们使用Valgrind中的工具集- callgrind进行内存泄漏的分析,然后通过kcachegrind进行图形化

```cpp
// code.c

#include <stdio.h>
#include <stdlib.h>
void *mem_alloc(int size);

void mem_free(void *mem_ptr);

int main(void)
{
    int size;

    printf("Please input the size to allocate memory: ");
    scanf("%d",&size);
    int *ptr = (int *)mem_alloc(size);

    return 0;
}

void *mem_alloc(int size)
{
    void *ptr = malloc(size);

    return ptr;
}

void mem_free(void *ptr)
{
    free(ptr);
}
```

上面的代码很明显的内存泄漏,我们通过valgrind的memcheck工具进行检测

![valgrind --tool=memcheck --leak-check=full ./a.out 
](http://p8uroi1uf.bkt.clouddn.com/valgrind--full_check.png)

另外我们还可以进行代码分析,如下:

![valgrind --tool=callgrind ./a.out && kcachegrind callgrind.out.*](http://p8uroi1uf.bkt.clouddn.com/valgrind_kcachegrind.png)

### 2. 总线错误(Bus Error)

这个错误,当时被很多人调侃过,说是"公交车错误"

*其实,我也就是在学校的OJ上见过那么一次,*

事实上,**这是RISC架构最容易出现的问题,内存的非对齐访问,会造成总线堵塞,进而引发此错误**

**然而,沃尔玛能通常是用的都是CISC的x86_64架构,事实上通过修正电路,降低一点效率,避免这个错误**

**但是,无论如何,我们都是时刻关心内存对齐,很关键**

在这个之外其实还有一种错误,"Segement fault" #=> "段错误"

在了解内存管理之前我对这个没有一点映像

**不过,现在看来,还是很关键的,这就是严重缺页错误,在分页机制中有三种映射错误**

**次要,严重,以及段错误,另外两种不多说,段错误就是映射失败,访问到不能访问的区域**

**不仅仅是TLB中没有要访问的页,内存中也没有,非法访问所造成的就是段错误,会造成程序中断进行**

**其实就是这么简单,Haha,多看点书还是有用的**

***

June 5, 2018 12:43 AM

*上面这些之后,就算咸鱼书彻底了结,C也了结了,当然kangkang的Esential C++看的也差不多,*

*就等C++ Primer(PS: 本来想叫C3P的,想像还是算了)就绪了.*

*虽然已经很晚了,不过,我有信心肝下去,干他妈的*

*想想后面的书: 6E, Server + Kernel 哇,脑子疼*

*但是,干他妈的总没错,溜了,溜了*
