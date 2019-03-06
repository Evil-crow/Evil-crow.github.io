---
layout: post
title: Linux x86_64系统调用的实现过程
date: June 7, 2018 10:10 PM
excerpt: 在看TLPI的时候,虽然重要的是系统调用的接口使用,但是系统调用到底是怎么一回事? 虽然有点小题大做,但是还是十分重要的,网上的大多数材料提到的也是Linux在ARM的分析实现,由于我自己使用的是x86_64, 所以还是来看看Linux Syscall的具体实现吧
categories:
- Unix/Linux
tags:
- Syscall
toc: true
comments: true
---

*由于系统调用的分析实现其中涉及一些内核/OS方面的知识,不是十分重要的,我们不进行详细说明*

*另外, 此分析,基于glibc-2.26以及Linux-Kernel-4.17,可以获取相关源码阅读*

从我们以往的理解上来看,系统调用就是使用OS提供的接口,不也就是调用么?

System Call涉及了用户/内核态的切换,有些是需要进入内核态完成的,所以系统调用的存在就很有必要

更重要的一点就是: **使用系统调用,仅仅暴露接口,保证了内核不会被无意/恶意破坏**

下面我们开始具体的系统调用实现分析:

## 一, 从使用一个系统调用开始

```cpp
#include <stdio.h>
#incldue <unistd.h>
#include <sys/types.h>
#include <fcntl.h>

int main(int argc, char *argv[])
{
    int fd = open("/tmp/file", O_RDONLY, mode_t mode);
	if (fd == -1)
		do_something(errno);
	
	close(fd);
	
	return 0;
}
```

上面便是一个简单地使用系统调用的例子,使用了open(系统调用)

那么下一步, **就是,open()函数的实现到底在哪里呢?**

有两种方法查看:

### 1. 使用GDB单步进入

这个时候,GDB的Step指令就十分重要了,对于open(),函数它可以单步进入啊!

这不就很明显的看出`open()`实现在哪里了?

![gdb step指令](http://www.qiniu.evilcrow.site/TLPI_gdb_s.png)

![locate 确定文件位置](http://www.qiniu.evilcrow.site/TLPI_locate.png)

如上

### 2. 合理推测(ಡωಡ)

我们使用的系统调用都是C实现,而在Linux平台下,使用最多的是GNU C,依赖的libc库为glibc

(PS: 上面是我的本机环境,各位可以根据自己的机器环境去C库进行实现的查找)

见我们的实现依赖于glibc,那么去C可查找理所当然了

然后使用查找关键字的方法就定位到open64.c上了

**此处,有一个巨坑,关于/usr/src/debug/, 这个里面glibc不全啊**

**只是为了调试需要,所集成的一部分glibc实现,同时需要注意我们使用的C库一般都是发行版提供**

**需要去看glibc源码才能真正看清楚**

## 二, 分析一个系统调用的源码

那么,现在就来刺激的了,分析系统调用的源码

```cpp
//sysdeps/unix/sysv/linux/open64.c

/* Copyright (C) 1991-2018 Free Software Foundation, Inc.
   This file is part of the GNU C Library.

   The GNU C Library is free software; you can redistribute it and/or
   modify it under the terms of the GNU Lesser General Public
   License as published by the Free Software Foundation; either
   version 2.1 of the License, or (at your option) any later version.

   The GNU C Library is distributed in the hope that it will be useful,
   but WITHOUT ANY WARRANTY; without even the implied warranty of
   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
   Lesser General Public License for more details.

   You should have received a copy of the GNU Lesser General Public
   License along with the GNU C Library; if not, see
   <http://www.gnu.org/licenses/>.  */

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdarg.h>

#include <sysdep-cancel.h>
#include <not-cancel.h>

#ifdef __OFF_T_MATCHES_OFF64_T
# define EXTRA_OPEN_FLAGS 0
#else
# define EXTRA_OPEN_FLAGS O_LARGEFILE
#endif

/* Open FILE with access OFLAG.  If O_CREAT or O_TMPFILE is in OFLAG,
   a third argument is the file protection.  */
int
__libc_open64 (const char *file, int oflag, ...)
{
  int mode = 0;

  if (__OPEN_NEEDS_MODE (oflag))
    {
      va_list arg;
      va_start (arg, oflag);
      mode = va_arg (arg, int);
      va_end (arg);
    }

  return SYSCALL_CANCEL (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS,
			 mode);
}

strong_alias (__libc_open64, __open64)
libc_hidden_weak (__open64)
weak_alias (__libc_open64, open64)

# if !IS_IN (rtld)
int
__open64_nocancel (const char *file, int oflag, ...)
{
  int mode = 0;

  if (__OPEN_NEEDS_MODE (oflag))
    {
      va_list arg;
      va_start (arg, oflag);
      mode = va_arg (arg, int);
      va_end (arg);
    }

  return INLINE_SYSCALL_CALL (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS,
			      mode);
}
#else
strong_alias (__libc_open64, __open64_nocancel)
#endif
libc_hidden_def (__open64_nocancel)

#ifdef __OFF_T_MATCHES_OFF64_T
strong_alias (__libc_open64, __libc_open)
strong_alias (__libc_open64, __open)
libc_hidden_weak (__open)
weak_alias (__libc_open64, open)

strong_alias (__open64_nocancel, __open_nocancel)
libc_hidden_weak (__open_nocancel)
#endif

```

*备注: 在本目录下,其实是有4个于open()有关的文件,分别是open.c,open64.c, openat.c, openat64.c*

*但是,事实上他们是指不同层次,不同平台上的系统调用,剥开之后,最后都是openat()函数*

*但是,我本机是64为,所以是封装的64位openat()函数文件,即open64.c*

我们可以一行一行分析这个函数:

```cpp
// function open() -> __libc_open64()

1 int
2 __libc_open64 (const char *file, int oflag, ...)
3 {
4   int mode = 0;
5
6   if (__OPEN_NEEDS_MODE (oflag))
7     {
8       va_list arg;
9       va_start (arg, oflag);
10      mode = va_arg (arg, int);
11      va_end (arg);
12    }
13
14  return SYSCALL_CANCEL (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS,
15			 mode);
16 }
17 strong_alias (__libc_open64, __libc_open)
18 strong_alias (__libc_open64, __open)
19 libc_hidden_weak (__open)
20 weak_alias (__libc_open64, open)
21
22 strong_alias (__open64_nocancel, __open_nocancel)
23 libc_hidden_weak (__open_nocancel)
```

*上面的C是典型的GNU风格,两个空格,看的我想打人,写惯了K&R C的人表示脑子疼*

6 ~ 8行就是简单的参数判断,我们可以略过

其实你可能会奇怪,好端端的`open()`,系统调用怎么变成了,`__libc_open64()`

**注意17 ~ 23行的宏,他们将别名实现为了宏,事实上,调用open(),就是__libc_open64()**

那么,重心来了,就是14行,它实现了,`SYSCALL_CANCEL`宏来进行操作

看到这里,这一部分文件就足够了,我们已经知道了`__libc_open64()`只是一个warpper function了

这便是TLPI中第一部分的,Warpper函数

## 三,Warpper Function如何Warpper

我们要去看warpper function如何工作,也就是去看它是如何进行封装,并且完成用户/系统态切换的

首先,Warrpper的过程,各个平台上都应该是一致的,所以我们去看,sysdeps/unix/sysdeps.h

这个文件中都是宏定义,那么,我们就一步一步来吧,我将宏定义的顺序整理了一下

```cpp
//sort the macros 已知 SYSCALL_CANCEL

1>
#define SYSCALL_CANCEL(...) \
  ({									     \
    long int sc_ret;							     \
    if (SINGLE_THREAD_P) 						     \
      sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__); 			     \
    else								     \
      {									     \
	int sc_cancel_oldtype = LIBC_CANCEL_ASYNC ();			     \
	sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__);			     \
        LIBC_CANCEL_RESET (sc_cancel_oldtype);				     \
      }									     \
    sc_ret;								     \
  })

+------------------------+
({ long int sc_ret; if (SINGLE_THREAD_P) sc_ret = INLINE_SYSCALL_CALL (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode); else { int sc_cancel_oldtype = LIBC_CANCEL_ASYNC (); sc_ret = INLINE_SYSCALL_CALL (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode); LIBC_CANCEL_RESET (sc_cancel_oldtype); } sc_ret; });
+------------------------+

2>
#define INLINE_SYSCALL_CALL(...) \
  __INLINE_SYSCALL_DISP (__INLINE_SYSCALL, __VA_ARGS__)

+------------------------+
__INLINE_SYSCALL_DISP (__INLINE_SYSCALL, openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode);
+------------------------+

3>
#define __INLINE_SYSCALL_DISP(b,...) \
  __SYSCALL_CONCAT (b,__INLINE_SYSCALL_NARGS(__VA_ARGS__))(__VA_ARGS__)

+------------------------+
__INLINE_SYSCALL__INLINE_SYSCALL_NARGS(openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode)(openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode);
+------------------------+

4>
#define __INLINE_SYSCALL_NARGS(...) \
  __INLINE_SYSCALL_NARGS_X (__VA_ARGS__,7,6,5,4,3,2,1,0,)

+------------------------+
__INLINE_SYSCALL__INLINE_SYSCALL_NARGS_X (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode,7,6,5,4,3,2,1,0,)(openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode);
+------------------------+

5>
#define __INLINE_SYSCALL_NARGS_X(a,b,c,d,e,f,g,h,n,...) n
// 这是最神奇的宏,一个计数宏,计算参数的宏

+------------------------+
__INLINE_SYSCALL4(openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS, mode);
+------------------------+

6>
#define __SYSCALL_CONCAT(a,b)       __SYSCALL_CONCAT_X (a, b)
#define __SYSCALL_CONCAT_X(a,b)     a##b
```

所以,同一处理后,获取到的是`__INLINE_SYSCALLn(name, arguments...); [n个参数]`

然后用同文件下的宏继续处理后,可以获取到 `INLINE_SYSCALL (name, nr, arguments...); `

既然我们分析的是x86_64平台上系统调用的实现,那么我们就去x86_64下找东西吧

下面找得到的就是具体的上述宏平台上的依赖实现了

```cpp
// sysdeps/unix/sysv/linux/x86_64/sysdeps.h

# define INLINE_SYSCALL(name, nr, args...) \
  ({									      \
    unsigned long int resultvar = INTERNAL_SYSCALL (name, , nr, args);	      \
    if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (resultvar, )))	      \
      {									      \
	__set_errno (INTERNAL_SYSCALL_ERRNO (resultvar, ));		      \
	resultvar = (unsigned long int) -1;				      \
      }									      \
    (long int) resultvar; })

/* Define a macro with explicit types for arguments, which expands inline
   into the wrapper code for a system call.  It should be used when size
   of any argument > size of long int.  */
# undef INLINE_SYSCALL_TYPES
# define INLINE_SYSCALL_TYPES(name, nr, args...) \
  ({									      \
    unsigned long int resultvar = INTERNAL_SYSCALL_TYPES (name, , nr, args);  \
    if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (resultvar, )))	      \
      {									      \
	__set_errno (INTERNAL_SYSCALL_ERRNO (resultvar, ));		      \
	resultvar = (unsigned long int) -1;				      \
      }									      \
    (long int) resultvar; })

#undef INTERNAL_SYSCALL
#define INTERNAL_SYSCALL(name, err, nr, args...)			\
	internal_syscall##nr (SYS_ify (name), err, args)

#undef INTERNAL_SYSCALL_NCS
#define INTERNAL_SYSCALL_NCS(number, err, nr, args...)			\
	internal_syscall##nr (number, err, args)

#undef internal_syscall0
#define internal_syscall0(number, err, dummy...)			\
({									\
    unsigned long int resultvar;					\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number)							\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

#undef internal_syscall1
#define internal_syscall1(number, err, arg1)				\
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1)						\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

#undef internal_syscall2
#define internal_syscall2(number, err, arg1, arg2)			\
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2)				\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

#undef internal_syscall3
#define internal_syscall3(number, err, arg1, arg2, arg3)		\
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);			 	\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;			\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3)			\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

#undef internal_syscall4
#define internal_syscall4(number, err, arg1, arg2, arg3, arg4)		\
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg4, __arg4) = ARGIFY (arg4);			 	\
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);			 	\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg4, _a4) asm ("r10") = __arg4;			\
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;			\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4)		\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

#undef internal_syscall5
#define internal_syscall5(number, err, arg1, arg2, arg3, arg4, arg5)	\
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg5, __arg5) = ARGIFY (arg5);			 	\
    TYPEFY (arg4, __arg4) = ARGIFY (arg4);			 	\
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);			 	\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg5, _a5) asm ("r8") = __arg5;			\
    register TYPEFY (arg4, _a4) asm ("r10") = __arg4;			\
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;			\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4),		\
      "r" (_a5)								\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

#undef internal_syscall6
#define internal_syscall6(number, err, arg1, arg2, arg3, arg4, arg5, arg6) \
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg6, __arg6) = ARGIFY (arg6);			 	\
    TYPEFY (arg5, __arg5) = ARGIFY (arg5);			 	\
    TYPEFY (arg4, __arg4) = ARGIFY (arg4);			 	\
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);			 	\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg6, _a6) asm ("r9") = __arg6;			\
    register TYPEFY (arg5, _a5) asm ("r8") = __arg5;			\
    register TYPEFY (arg4, _a4) asm ("r10") = __arg4;			\
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;			\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4),		\
      "r" (_a5), "r" (_a6)						\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})
```

**此文件内容超多,所以我们只截取其中重要的部分**

我们可以看到下面这样的实现流程:

`INLINE_SYSCALL(name, nr, args...)` -> `INTERNAL_SYSCALL(name, ,nr, args)` ->

`internal_syscall##nr (SYS_ify (name), err, args);`

下面就是具体对应不同参数的函数实现了,其中实现了,产生软中断,进入内核态,然后参数入寄存器,传参

那么,这一些列函数中我们可以看到:

**参数只能是6个,可以传递7个参数,一个是系统调用名,是吗? 错的,传递的参数只是建议范围内为7个**

**其余的呢? 入栈, Register参数就7个罢了**

我们来具体看看函数实现,尤其是其中的内联汇编部分(其实我汇编很烂,Intel都不行,ATT就更不多说了)

```x86asm
asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4),		\
      "r" (_a5), "r" (_a6)						\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);
```

其他的不多说,其中最重要的就是, `"syscall\n\t"`这个东西了

> 在32位机上,使用`int 0x80` 第128中断号,陷入内核,进行系统调用

> 在ARM上,使用`swi `执行陷入内核

> 唯独少有人说x86_64平台上的软中断,也就是`syscall`中断向量

你没有听错,这个B就是叫syscall中断vector,qtmd,看了我半天

*不过这也是我内核基础不扎实的恶果,深感惭愧*

至此,Warpper function的任务顺利完成,下来就需要切换到内核态处理了.

我们接下来就得看LinuxKernel的代码了

## 四,关于中断的小了解

中断是在引入保护模式(protect mode)之后很重要的一个特性,以此来陷入内核,进入内核态

**一般的fault,会重复指令,syscall会vector+1, 执行下一条**

*此处,中断的内容,同样是Kernel重要知识,当然我还是不够清楚,不能清晰地翻出汇编指令解析*

*但是,syscall的流程还是比较清楚的*

首先,来看内核的中断处理例程

既然是64位,肯定看arch/x86/entry/entry_64.S

简单说一下,entry这几个文件

> entry_32.S是32位机使用的

> entry_64.S是64位机使用的,但是说实话,很像是拼凑的,凌乱不堪,就在32基础上改得,32看着十分清晰

> entry_64_compat.S 是兼容的,在64位机器上按照32的形式运行,用的是32位的接口

```x86asm
ENTRY(entry_SYSCALL_64_trampoline)
	.....
	movq	$entry_SYSCALL_64_stage2, %rdi
	JMP_NOSPEC %rdi
END(entry_SYSCALL_64_trampoline)

	.popsection

ENTRY(entry_SYSCALL_64_stage2)
	UNWIND_HINT_EMPTY
	popq	%rdi
	jmp	entry_SYSCALL_64_after_hwframe
END(entry_SYSCALL_64_stage2)

ENTRY(entry_SYSCALL_64)
	UNWIND_HINT_EMPTY
	

	swapgs
	
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp

	/* Construct struct pt_regs on stack */
	pushq	$__USER_DS			/* pt_regs->ss */
	pushq	PER_CPU_VAR(rsp_scratch)	/* pt_regs->sp */
	pushq	%r11				/* pt_regs->flags */
	pushq	$__USER_CS			/* pt_regs->cs */
	pushq	%rcx				/* pt_regs->ip */
GLOBAL(entry_SYSCALL_64_after_hwframe)
	pushq	%rax				/* pt_regs->orig_ax */

	PUSH_AND_CLEAR_REGS rax=$-ENOSYS

	TRACE_IRQS_OFF

	/* IRQs are off. */
	movq	%rax, %rdi
	movq	%rsp, %rsi
	call	do_syscall_64		/* returns with IRQs disabled */
	
	.........
END(entry_SYSCALL_64)
```

其中删去了很多内容,但是整个思路是:

1. 去中断向量表查看中断偏移量(很惭愧,没找到),

2. 对应的entry_64.S进行section的调用

3. 对于syscall就是调用到了(其中还有两个函数被我删掉了,仅保留了核心)`entry_SYSCALL_64`

 *其中ENTRY(), END(),都是宏用来进符号表中的函数声明*

4. 在`entry_SYSCALL_64`中调用了`do_syscall_64()`

`do_syscall_64()`这是一个外部函数,在common.c中的定义

```cpp
// arch/x86/entry/common.c

__visible void do_syscall_64(unsigned long nr, struct pt_regs *regs)
{
	struct thread_info *ti;

	enter_from_user_mode();
	local_irq_enable();
	ti = current_thread_info();
	if (READ_ONCE(ti->flags) & _TIF_WORK_SYSCALL_ENTRY)
		nr = syscall_trace_enter(regs);

	/*
	 * NB: Native and x32 syscalls are dispatched from the same
	 * table.  The only functional difference is the x32 bit in
	 * regs->orig_ax, which changes the behavior of some syscalls.
	 */
	nr &= __SYSCALL_MASK;
	if (likely(nr < NR_syscalls)) {
*		nr = array_index_nospec(nr, NR_syscalls);
*		regs->ax = sys_call_table[nr](regs);
	}

	syscall_return_slowpath(regs);
}
```

其中标上*的两句就是最重要的,它会去sys_call_table[]全局例程调用表中查找我们想要调用的例程

为什么有这样的机制呢?

因为中断号页只有255,所有的系统调用共享syscall对应的入口,所以我们需要一个数组

**在都进入内核态之后,根据每个系统调用的唯一编号来进行唯一标识与识别**

这个编号的确定,是有各种各样的宏确定的,**同时要保证LinuxKernel与glibc的编号一致,**

我们可以来看这两个文件

```cpp
// /usr/include/asm/unistd_64.h

#ifndef _ASM_X86_UNISTD_64_H
#define _ASM_X86_UNISTD_64_H 1

#define __NR_read 0
#define __NR_write 1
#define __NR_open 2
#define __NR_close 3
#define __NR_stat 4
#define __NR_fstat 5
#define __NR_lstat 6
#define __NR_poll 7
#define __NR_lseek 8
#define __NR_mmap 9
#define __NR_mprotect 10
#define __NR_munmap 11
#define __NR_brk 12
#define __NR_rt_sigaction 13
#define __NR_rt_sigprocmask 14
#define __NR_rt_sigreturn 15
#define __NR_ioctl 16
#define __NR_pread64 17
.........
```

```cpp
// ....
```

确定好编号后,就可以访问指定函数的地址了

## 五,真正的系统调用实现

我们可以查看此文件

```cpp
// /include/linux/syscalls.h
// 此文件实现了所有系统调用的声明以及SYSCALL_DEFINEx宏

#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

#define __SYSCALL_DEFINEx(x, name, ...)					\
	asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
		__attribute__((alias(__stringify(__se_sys##name))));	\
	ALLOW_ERROR_INJECTION(sys##name, ERRNO);			\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
	
			.........
	/* fs/eventfd.c */
asmlinkage long sys_eventfd2(unsigned int count, int flags);

/* fs/eventpoll.c */
asmlinkage long sys_epoll_create1(int flags);
asmlinkage long sys_epoll_ctl(int epfd, int op, int fd,
				struct epoll_event __user *event);
asmlinkage long sys_epoll_pwait(int epfd, struct epoll_event __user *events,
				int maxevents, int timeout,
				const sigset_t __user *sigmask,
				size_t sigsetsize);

/* fs/fcntl.c */
asmlinkage long sys_dup(unsigned int fildes);
asmlinkage long sys_dup3(unsigned int oldfd, unsigned int newfd, int flags);
asmlinkage long sys_fcntl(unsigned int fd, unsigned int cmd, unsigned long arg);
#if BITS_PER_LONG == 32
asmlinkage long sys_fcntl64(unsigned int fd,
				unsigned int cmd, unsigned long arg);
				
				.......
```

经常看到其他文档上说到sys_open()是入口,这其实都是早期的说法,实际上,是由宏生成真正的函数定义

同时我们可以根据,此文件中的注释,找到根本意义上的Syscall定义,比如`epoll_ctl()`

```cpp
/*
 * The following function implements the controller interface for
 * the eventpoll file that enables the insertion/removal/change of
 * file descriptors inside the interest set.
 */
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int full_check = 0;
	struct fd f, tf;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
	struct eventpoll *tep = NULL;

	error = -EFAULT;
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;

	error = -EBADF;
	f = fdget(epfd);
	if (!f.file)
		goto error_return;
................
```

**上述即为真真正正最底层,最底层`epoll_ctl()`的实现**

**至此,我们便真正追踪到了,系统调用的实现了,其他系统调用的源码,查看方法同理**

## 六,扩展阅读

[关于ARM实现](https://my.oschina.net/fileoptions/blog/908682)

[关系不大,但有价值的glibc分析(脚本Warpper)](https://zhuanlan.zhihu.com/c_144857088)

书籍推荐: < Linux内设设计与实现 >        (PS: 虽然是基于2.6.32,但还是很有意义)

## 其他的一些闲杂事情

主要还是最近烦,感觉自己要玩了(beibei狗),看TLPI,突发奇想,想看看syscall的真正实现

但是网上都是ARM,X86是实现(2.6.32),没有X86_64,4版本内核的实现

于是乎我决定来稿一搞

但是自身水平又不行,所以也比较磕绊,其实分析也是不完整的

最后,系统调用的退出,如何添加自己的系统调用这些内容都是不全面的

**还是自己学习不扎实,后面落实后会补充的,起码这两天干的事,还是挺有意义的**

*接下来,回归主要任务,TLPI作为辅修,是时候C3P就绪了*

**虽然很烦,但还是那句话:**

> 生命转瞬即逝, 没时间丧,Fighting!

June 8, 2018 2:00 AM
