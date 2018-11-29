---
layout: post
title: VimScript的学习(二)
date: October 18, 2017 12:30 AM
excerpt: 最近对于VS的学习真的是着了魔,
categories:
- Vim 
tags:
- VimScript
comments: true
---

## 编辑Vimrc配置文件

有的时候会发现一些很容易提高效率的技巧

但是,且切来切去的打开Vimrc会打断你的思路

那么,怎么办呢?

设置特殊映射即可

```
let mapleader=","
nmap <leader>ev :vsplit $MYVIMRC<cr>
name <leader>sv :source $MYVIMRC<cr>
```

即可通过使用简短的快捷键完成**修改**并且**重新加载**vim的配置文件的功能

使用变编辑的这两个映射

可以快速完成,映射,加载的功能

# Abbreviations(缩写)


在Vim这么强大的编辑器面前,有着各种各样的方便特性

比如说,我们现在要谈到的缩写,

**缩写**,这个特性的作用类似于**映射**

但是,与映射又不尽相同,

**因为映射逐一扫描,不会考虑上下文环境,直接替换**

**但是,缩写会考虑上下文环境,进行合理的处理**

一般情况下,缩写用于:

- 解决平时一些经常出现的小问题,比如我自己经常写incldue

- 处理一些,因为键位的问题,比如@fujie的:Q

而且,缩写有三种模式,替换模式暂不讨论

```
iabbrev 关键词 替换内容 (insert mode)
cabbrev 关键词 替换内容 (command mode)
```
输入要替换的内容后,直接空格就会进行替换

或者,输入 < cr > 也是可以进行缩写替换的

更多的时候,我们还可以使用"缩写替换"的功能来进行文本标签的书写

```
iabbrev @@ Evilcrow486@gmail.com
iabbrev ccopy Copyright.....
```

很方便的功能,不是吗?

### 更强大的Mappings
---
如果要使自己进步就需要做出一些改变

首先,从改掉之前的快捷键开始

```
noremap <esc> <nop>
nnoremap jk <esc>
```

首先从禁止掉< Esc >及一系列的方向键开始

强迫自己习惯新的,高效的,工作习惯

# 本地缓冲区及设置
这里,要钱车道的一个概念就是:本地缓冲区

即之前常用到的local

**在,Mappings,Abbreviations,Options中可以使用< buffer >来指定缓冲区**

**在leader中,可以使用< localleader >*,简单来讲,就是一个本地的,局部变量的问题**

**第三点,某些布尔选项也是支持setlocal的**

**最后强调一点,相对而言,本地的设置会覆盖全局的,因为本地的更为具体**

## 自动命令(Auto-command
**映射,缩写,自动命令是使Vim可以高度定制化必不可少的强力工具**

自动命令简单的讲:**在Vim中触发某种事件的时候,让Vim去执行提前规定好的内容**

举个例子:

```
autocmd BufNewFile * :write
```

这个例子,可以这样解释:

当Vim中**新文件创建**的事件触发时,执行 :w进行保存

(这些文件是任意文件,*进行匹配)

总结一下:

```
autocmd + 事件 + 匹配 + 操作处理
```
**这解决了,我们经常处理到的,Vim在第一次保存前,并没有实际创建文件的问题**

同时,autocmd可监控多个事件

**多个事件使用 , 进行分隔**

同时,Vim中有个不成文的规定,

所有的文件应该用**BufRead与BufNewFile**一起打开

```
autocmd BufNewFile,BufRead *.c setlocal nowrap
```
这样,不论何时,打开的C文件,都不能自动卷曲换行

**而在所有的事件中,最有用的就莫过于FilType文件了**

FilType文件会自动识别文件类型,然后执行autocmd

**结合FilType事件与< buffer >,或者local的设定,就很棒,可以针对不同的缓冲区进行不同的配置,Perfect!**

**切记,切记,使用FileType事件时,一定不要大写,一定要时小写! ! ! **
