---
layout: post
title: VimScript的学习(三)
date: October 18, 2017 7:23 PM
excerpt: 这一篇是关于配置篇的最后一篇了
categories:
- Vim 
tags:
- VimScript
comments: true
---

## 自动命令组

VimScript中多次使用自动命令,会进行重载,

并且将命令进行多次使用,叠加,会严重影响Vim的速度

所以,使用自动命令组,进行合理配置

```
augroup testgroup
  autocmd BufWrite * :echom "hello"
  autocmd BufWrite * :echom "Vim"
augroup END
```

但是,使用自动命令组,还是会进行命令的叠加

肯定有办法解决

使用 autocmd! 清除自动命令组

**可以避免进行自动命令组的重载**

```
augroup testgroup
  autocmd!
  autocmd BufWrite * :echom "hello"
  autocmd BufWrite * :echom "Vim"
augroup END
```

这样可以定义多组自动命令,且不会重复加载

## Opearate-Pending

mapping 是我们之前提到过,VimScript功能强大的一个很重要的原因

之前提到了,`map`,`imap`,`vmap`,`nmap` 以及用`nore`表示非递归

但是,还有一种十分重要的mapping

**Operate-Pending** 表示**在一个操作之后,跟着进行移动的命令,**

**此操作会从开始的位置,一直到移动结束的位置为止**

常用的操作符及常见的移动命令

|按键|操作|移动|
|----|---|---|
|dw|删除|直到下一个单词|
|ci(|修改|在括号内|
|yt,|复制|到逗号|

这一部分,我们要进行的不是修改操作的映射,而是修改移动的映射

即**Movement的映射**

```
:onoremap p i(
```
即将p键映射到在括号内修改

再来看一个例子

```
:onoremap b /return<cr>
```

b键现在映射为,搜索到的所有的return

例如执行`db` 则会删除所有的`return`

当然,Vim这种可定制化程度高的编辑器

一定是可以**修改光标开始的位置**

```
:onoremap in( :<c-u>normal! f(vi(<cr>
```

将`in(`映射为,`f(vi(<cr>`

**其中的< c-u >与normal!,会在之后解释,但是目前与映射的功能无关**

**f( -> 向下寻找并移动到括号**

**v i ( 进入可视化模式,并且选中括号内所有内容**

所以,这条映射的作用是,向下寻找并且选中括号内所有内容

同理,向上寻找为:

```
:onoremap il( :<c-u>normal! F)vi(<cr>
```

这两个命令,可以理解为`i(`的强化与拓展

一般规则如下,可以更清晰的明白如何进行**Operate-Pending**如何工作

**1. 如果你的Operator-pending映射在可视化模式下选中文本结束,Vim会操纵这些文本**

**2. 否则Vim会操作从光标的原始位置到一个新位置的文本**

## 更多,强大的Operator-Pending映射

现在开始,我们进行更为强大的Vim的映射

```
:onoremap ih :<c-u>excuate "normal! ?^==\\+$\r:nohlsearch\rkvg_"<cr>
```

第一眼看上去,这个映射也使我头大,那么有什么办法呢?

继续进行拆分

**excuate 表示执行后面作为Vim脚本字符串部分的命令**

**那么,excuate,normal!都可以表示执行,有什么区别,**

**实质上,normal!,是不能识别转义字符的,反之excuate是可以的**

**当excuate遇到想要执行的脚本字符串的时候,首先会替换特殊字符**

```
"noamal! ?^== \ +$ <换行> :nohlsearch <换行> kvg_" <cr>
```

**?^==+$,表示搜索任何以2个以上== ,且只有'='的行,并且切换到行首**

**:nohlserach,取消搜索的语法高亮**

**kvg_ ,移动到上一行,可视化模式,移动到本行最后一个非空字符上,不使用$,是因为不需要选中'\0'**

**这一个Operate-Pending的作用就是选中,向后的一个,标题行**

```
:onoremap an :<c-u>execute "normal! ?^==\\+\r:nohlserach\rg_vk0"<cr>
```

相比与上一条,改变之处在于:

```
\rg_vk0"<cr>
```
选中到最后一个非空字符,之后进入可视化模式,向上移动一行,再将光标移动至行首,

**实现了选中,上下两行的功能**

## 状态条

平时经常看见,下面的状态条,那么,这也能定制化吗? ? ?

**当然可以**

使用`statusline`来进行定制

```
:set statusline=%f   查看编辑路径
:set statusline=%f\ - \ FileType:\ %y
```
VimScript在此处,空格需要进行转义

这其中的格式化字符串,是类似于C以及Python的

但是,上面那样的写法,可读性是十分垃圾的,所以我们建议:

**使用叠加的方式来写**

```
:set statusline=%f          " 文件名即及路径
:set statusline+=\ -\       " 空格
:set statusline+=FileType:  " 标签
:set statusline+=%y         " 类型名
```
statusline的设定可以使用各式各样的,修饰符,如何操作,由使用者自行决定


## 负责任的编码

VimScript作为一门十分小众的语言,但是他也存在程序员必须遵守的编程准则

**注释**

**分组**

*使用其中的代码折叠功能,使得你的配置可以清晰的展现出来*

**简写**

**强烈要求:不要使用简写,会使你的代码混乱不堪**

**至此,第一部分进行Vimrc的基础配置到此结束,下面开始,会将VimScript作为一门真正的编程语言来学习**

October 19, 2017 2:31 AM
