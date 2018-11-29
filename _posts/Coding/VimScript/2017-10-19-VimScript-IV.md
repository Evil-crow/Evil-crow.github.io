---
layout: post
title: VimScript的学习(四)
date: October 19, 2017 3:05 AM
excerpt: 现在开始,正式是编程的脚本语言VimScript的学习,以后可是可以开发插件的
categories:
- Vim 
tags:
- VimScript
comments: true
---

*说在前面*

*选择VimScript学习,真的是一腔热血,其实这门语言的局限型*

*比想象中还要小,不只是只能用于Vim插件的开发*

*而且,是只能跑在Vim平台上的,`:source`,执行脚本,以`.vim`为文件扩展名*

*而且,因为小众,历史原因的问题,语法也有点混乱,算是微型PHP*

*其中有着各种各样的奇奇怪怪的问题,很烦人的*

*但是,兴趣既是导向,学吧,Fighting!*

**也感谢@kang,让我明白.想做就去做**

## 关于VimScript

**这是一门将脚本语言 + Vim设置 混搭的语言**

**只能运行在Vim平台上的**

**在Vim中 : source  * . vim 即可运行VimScript程序**

**文件扩展名为 .vim **

**另外注明 : 在VimScript中进行书写的时候,是不需要 : 的,而含有":"的都是在Vim中的命令操作**
## 变量

VimScript作为一门脚本语言,其中也是可以使用变量的

来看一个简单地例子

```
:let foo = "bar"
echo foo  #=> "bar"
:let foo = 5
echo foo  #=> 5
```

从上面可以看出,VimScript貌似是动态语言,但是,我只能说你太年轻了

后面的东西会告诉你,VimScript,**只能算是半动态语言**
### 作为变量的选项

VimScript是一门用于Vim脚本开发的脚本语言,所以,VimScript中既有脚本语言的内容,同时也嵌套着Vim命令

那么,Vim中作为设置选项的,可以作为变量使用吗?

**答案是肯定的**

对于作为变量设定的选项,也是可以作为变量使用,同时可以藉此进行选项的修改,利用脚本操作Vim

```
:echo &numberwidth
:let &numberwidth=10
:echo &numberwidth
```
同时,我们不仅可以设置数值型选项

布尔型选项也可以设置

**truthy = 所有非0值 , false = 0**

那么为什么要使用let呢?

```
:set textwidth=100
:let &numberwidth=&textwidsth-90
:echo &numberwidth
```

**即可看出,let是可以操作变量的.但是,set是只能操作数值型选项的,不能进行变量值的设定**

**在脚本语言中,使用let是更为方便的**

**但是,更为实际的建议是,能用set的时候就用set,因为let会使语法变得复杂**

### 本地选项

和之前提到过的进行设置是一样的,变量也是有作用域的,可以进行本地的设置

```
:let &l:number = 1
:echo &l:number

切换窗口
:let &1:number = 0
(第一个窗口无效)
```
联想之前进行的操作,相信你很容易理解,作为本地选项的含义和意义

### 将变量作为寄存器

VimScript一个强大的地方在于:**可以将寄存器作为变量使用**

```
:let @a = "hello"
:echo @a
```

同时,`"ap`可以将寄存器中的内容进行粘贴

有的寄存器是匿名的,"用来表示没有进行命名的寄存器

比如:

```
echo "@
```
在文件中选中一个单词并且进行复制,之后再进行输出

则这个寄存器就是未命名的

可以使用`echo @"`进行输出

**即使用@表明是使用寄存器,之后为寄存器名**

**:help registers中显示了可以使用的寄存器名**

## 变量的作用域

*了解过Python以及Ruby之后,会大体了解变量作用域的问题*

*我们再来看看Ruby作用域*

- 局部变量
- **$**全局变量
- **@@**类变量
- **@**实例变量

**目前,当变量是一个字母与一个冒号开头的时候,它就是一个作用域变量**

## 条件语句

VimScript中的条件语句,类似于Ruby,但是其中没有**unless**

if语句的语法为

```
if 条件1
  处理1
elseif 条件2
  处理2
else
  处理
endif
```

VimScript使用以上语法进行分支操作

同时,if语句的判断条件支持表达式expr

下来我们来看三个例子:

```
:echo "hello" + 10
:echo "10hello" + 10
:echo "hello10" + 10
```

结果依次是:10,20,10

很奇妙,对吗?这就是VimScrript比较恶心的地方了

**1. 字符串在进行 + 运算时,会强转为数值类型**

**2. 字符串强转时,以数字开头的字符串被强转为数值,否则为'0"**

## 比较(条件表达式)

**比较是很重要的一个东西**

**比较的结果,决定了条件语句,循环语句等等的结果,所以,比较语句是必须的**

**而在VimScript中,比较语句也是十分的恶心**

VimScript中,比较随处可见的,但是因为与Vim命令混用

所以,其中不乏会出现这样的东西:

```
:set ignorecase              " 设置大小写不敏感
:set noignorecase            " 设置大小写敏感
```

而这样,就给我们的比较带来了问题

**进行字符串的比较时,到底是大写,还是小写,怎么选择?这不就是糊弄人么!**

**所以,VimScript中提供了另外两个比较运算符**

** ==? 与 ==# 分别表示比较敏感与不敏感**

**这样的话,就不会出现比较前,需要注意一下,是否被人设置了敏感大小写与否**

## 函数(function)

作为一门编程语言,是一定要支持函数的,没错吧?

VimScript中的函数写法是类似于Ruby,Golang的,

```
function Hello( )
  echo "This is an function that is used for testing!\n"
  echo "Succeed!\n"
endfunction
call Hello( )
```
但是,他是没有Ruby中~ { ~ } ~ 块的写法的

其中严格要求使用

function ~ endfunction的形式进行函数的定义

而且,注意了!

**VimScript中没有函数作用域型限制的func建议使用大写字母开头**

上面我们则愉快的定义了一个名为Test的无作用域限制的函数

同时,我们在看看这个函数运行的结果

![](http://oww4cv296.bkt.clouddn.com/VimScript%284%29hello%28%29.png)
![](http://oww4cv296.bkt.clouddn.com/VimScript%284%29hello_result.png)

此处有一个深坑,就是进行这些函数测试的时候,一般都是写在Vim中的,

**因为写在.vim文件中,无法进行执行,source文件也并不能直接执行,source只是进行代码的运行**

**即,要使用命令行写,就写到底.或者在.vim文件中直接进行调用,不然在Vim中是不能直接调用的**

### 调用函数

在VimScript中,调用函数有两种方式,

**一,使用call 进行函数的调用,但是这个函数只能显示函数的副作用,所以对于没有副作用的函数,无效果**

**二,在表达式中进行函数的调用,不需要使用call,只要是使用函数的名字即可,同时传递函数的返回值**

举个例子,上面的Hello( )函数

可以使用`echom Hello( )`调用

**这会调用Hello( ),并将Hello( )的返回值传递给`echom`,但是Hello( )无返回值**

### 隐式返回

VimScript中,使用表达式进行函数的调用时,会有**返回值**,对于没有显示设定返回值的函数,会隐式返回0

```
function Hello( )
  echo "This is an function that is used for testing!\n"
  echo "Succeed!\n"
endfunction
echom Hello( )
```
![](http://oww4cv296.bkt.clouddn.com/VimScript%284%29_shadow_value.png)
例如,上面的代码执行后,会有一行为0

而进行这样的修改后

```
function Hello( )
  echo "This is an function that is used for testing!\n"
  echo "Succeed!\n"

  return "Return_Value"
endfunction
echom Hello( )
```

![](http://oww4cv296.bkt.clouddn.com/VimScript%284%29_result.png)

如果有C语言的编程经验,对于返回值的问题还是比较容易理解的

**如果VimScript函数不返回一个值,则它会隐式返回0**

### 函数参数

作为编程语言中的函数,**一定是可以接受参数的**

```
funcation DisplayName(name)
  echo "hello,my name is"
  echo a:name
endfunction
call DisplayName("Crow")
```

即可将字符串传入函数DisplayName(name)中,从而显示输入的字符串

**谨记:name在该函数中,是具有函数作用域的.所以,我们需要在前面加上 a: 来进行变量作用域的限制**

**如果,没有作用域限制,会变成全局作用域,VimScript会报错,无法找到变量**

**所以,在写需要变量的VimScript函数时,请总在变量前加上`a:`来显示变量的作用域**

### 可变参数

有的时候,不想规定参数个数,根据需要进行参数个数限定

可以使用可变参数进行函数的设计

```
function! Varg(...)
  echo a:0
  echo a:1
  echo b:2
  echo a:000
endfunction
call Varg(1,2)
```

**首先注意一点,我们从这里开始使用`function!`来进行函数的定义,为什么?**

**使用function! 可以覆盖之前进行加载过的函数,对于重名函数进行覆盖,所以,建议加上 ! 更为安全**

**否则,有时对于重名函数会进行报错,同时也建议使用 ! 的时候,不要使用重名函数,会进行覆盖.**

现在,我们来看看可变参数如何使用

**echo a:0 显示了可变参数的个数**

**echo a:1 显示了第一个参数**

**echo a:2 显示了第二个参数**

**echo a:000 则将可变参数转换成为列表**

**列表,同Python的列表,Ruby的数组**

同时,我们还可以将可变参数与普通参数一起使用,可以写出下面这样的函数

```
function! A(a,...)
  echo a:a
  echo a:0
  echo a:1
  echo a:000
endfunction
call A("Crow",1)
```

![](http://oww4cv296.bkt.clouddn.com/VimScript%284%29_varg.png)

VimScript会自行进行判断,处理好可变参数和具名参数的赋值

## 循环

三(四)大语句中,循环可是占了不小的分量.

但是,为什么不进行循环的介绍呢?

因为VimScript中,对于循环的需求是很小的,但是还是有了解的必要

**循环有:for循环,2 while循环两种**

### for循环

类似于Ruby中的for循环,从C转换过来,可能不是很适应

```
let foo = [1,2,3,4,5]
for i in foo do
  echo i
endfor
```
上面的循环,逐次遍历输出foo列表中的元素

### while循环

while的循环是与其他语言大同小异的

```
let foo = [1,2,3]
let i = 0
while i <= 2
  echo foo[i]
  let i = i + 1
endwhile
```

**需要注意,endwhile,endfunction,endfor不能写错**

**同时,对于进行值改变时,一定要加上let,进行操作**

_ _ _

最后,来谈谈echo与echom的区别吧

echo系的的函数有很多,的那是我们常用的就是echo与echom

其区别在于:

**echo: 会将结果进行结算,并且显示完后结束**

**echom: 将内容作为一个真正的消息进行显示并存储,所有消息必须时数值/字符串**

**列表,字典,浮点数这些都是不被允许的**

一般情况下,如果不是为了进行消息的存储,**建议使用 echo **

October 24, 2017 10:11 PM
