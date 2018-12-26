---
layout: post
comments: true
title: Linux的常用配置
date: July 31, 2018 4:14 PM
excerpt: 用了这么久的Linux,也稍稍记录一下自己的配置吧,以免以后需要
categories:
- Unix/Linux
- Guide
toc: true
---

*这是一篇自己给自己看的使用指南, 还是认真的记录一下比较好*

## 一, 基础内容

这一部分是每一个Linux安装成功之后,所要进行的合理操作

这里,针对我的机型,我用Fedora来举例子, 当然基本上是通用的redhat配置

### 1. 添加自己的sudo权限

作为Redhat系的传统,不提供sudo权限的添加,需要自己去进行修改.

Debian系设置好sudo权限,之后自行passwd

具体操作:

```bash
sudo vi /etc/sudoers
/ALL
(多按几次n, 找到ALL,有root的地方)
a
(仿照上面进行自己用户名的添加)
Esc -> :wq!

退出终端(刷新一下, OK!)
```

### 2. 添加合理的软件源

作为Fedora用户,着重推荐这两个源, FDZH以及RPMFusion的源

```bash
dnf install https://repo.fdzh.org/FZUG/free/27/x86_64/noarch/fzug-release-27-0.2.noarch.rpm
RPM Fusion的源需要去官网下载repo,之后使用rpm安装即可
```

[RPMFusion>>](https://rpmfusion.org)

[FDZH>>](https://github.com/FZUG/repo/wiki/添加-FZUG-源)

其他的源,根据自己的使用情况进行配置

添加好源之后,进行软件源的获取缓存

```bash
sudo dnf makecache
```

### 3. 快捷键的设置

此处,对于gnome可以按照这样的顺序:

- setting -> device -> keyboard -> add shortcuts

对于KDE,可以这样来设置

![setting shortcuts](http://www.qiniu.evilcrow.site/KDE_shortcuts.png)

### 4. 关闭神油SElinux

我基本也没用过, 反正就是个神油玩意

```bash
sudo vi /etc/selinux/config
"SELINUX=enabled" -> "SELINUX=disabled"
:wq!
```

### 5. 善用alias

在~/.bashrc中善于添加命令别名

```bash
alias aliyun_Crow='ssh Crow@ww.xxx.yyy.zzz'
```

上面的命令别名配上,ssh免密码登录简直不要太nice

另外就是自己写的几个小脚本,在其他地方也使用了,做些合理性判断,挺好用的

## 二, KDE的一些设置

> 目前最期待KDE plasma 5.13了,终于可以方便的改屏保,加速进入桌面的速度,减少内存开销

> 不过说好的6.12正式版推送,Fedora目前也没有啥动静,心累.(不过一段时间内,是不会更新28了)

> (还好,说这话的时候,29的Alpha版还没出,差点打脸)

这部分就是专门针对于KDE的设置了,用gnome的同学,我可能提供不了多少帮助了

不过有两点:

1. gnome使用 `sudo dnf install gnome-tweak-tool` 超级好用

2. gnome下IDE字体显示还是很好看的,KDE简直看不了,逼得我用Vim + VSCode

下面是对于KDE的完整配置方案,包括了Konsole呦(就那个初始丑的不像样子的终端)

... (挖个坑,保证一周内填上去)

--- 

update: July 31, 2018 11:33 AM

~~好吧,汪汪汪...~~

首先,我们从setting界面来入手

### Appearance

#### Workspce Theme

##### Look And Feel

使用Fedora主题, Breeze(*emmm,其实区别不大,但是我们毕竟是使用Fedora嘛*),

Breeze Dark则是暗色调, 个人不习惯.

这一处,Fedora基本符合我的口味,主要影响plasma, 颜色主题, 窗口和鼠标主题,

,喜欢的话,右下角还有 `Get New Looks...`按钮,可以自行挑选,商店就那样子...

![Look And Feel](http://www.qiniu.evilcrow.site/KDE_LookAndFeel.png)

##### Desktop Theme

这部分设置主要体现在任务栏的显示上. 见下:

![Air](http://www.qiniu.evilcrow.site/KDE_air.png)

![Dark](http://www.qiniu.evilcrow.site/KDE_dark.png)

![breeze](http://www.qiniu.evilcrow.site/KDE_breeze.png)

##### Cursor Theme

这部分设置主要是鼠标指针的设置,自行适配即可.

![Cursor Theme](http://www.qiniu.evilcrow.site/KDE_CursorTheme.png)

##### Splash Screen

这个很简单,就是开机进入的动画

*不是开机的壁纸,仅仅只是输入密码后,KDE Loading期间显示的东西.*

*但是,5.13听说会加速进入桌面的时间...*

#### Colors

此处没有子选项,而且变化不大.这里的颜色选项.主要是用来控制Dialog的色彩的.还可以自行再进行编辑

![Color Scheme](http://www.qiniu.evilcrow.site/KDE_ColorScheme.png)

#### Fonts

不过说了,**就是控制字体的地方**

其中主要影响的是: 

- **General(Software中的字体)**

- **Small(鼠标放上去的浮动字体)**

- **Toolbar(工具栏字体)**

- **Menu(菜单栏字体)**

- **Window title(窗口标题字体)**

**Warning:** 虽然我是KDE粉丝,但是**KDE的字体真的难看到模糊,尤其是软件中,IDEA,Pychrm,CLion**

这些都是重灾区,反观,不套用系统字体的编辑器玩的好好的,**Consolas,在KDE中默认渲染不是等宽**

*不过也不是没有办法,我们后面再聊*

*我的字体设置也不是很好看,但是还凑合能看,诸君可做参考*

![Fonts](http://www.qiniu.evilcrow.site/KDE_Fonts.png)

#### Icons

简单粗暴,就是进行图标的设置,怎么说呢,这个看个人喜好了.推荐去: 

1. [KDE Store >>](https://store.kde.org/)

2. [Gnome-look >>](https://www.gnome-look.org/browse/ord/latest/)

虽然两个不是同一家,这些个性化内容都是通用的.

个人比较推荐 [plane>>](https://store.kde.org/p/1178976/) 这一套图标的,(*VSC很好看,除了chrome中间的空洞勉强接受,其他完美*)

![Window Decorations](http://www.qiniu.evilcrow.site/KDE_WindowsDescl.png)

#### Application Style

其中还有另外三个子选项:

##### Widget Style

##### Window Decorations

##### GNOME Application Style(GTK)

第一个没啥意思,基本不管,第三个是处理GTK系列应用程序在KDE上的应用风格(有需要的童鞋摸索一下)

我一般用原生的KDE应用,不需要GTK应用,\滑稽.

我们重点来看第二个,Window Decoration

**主要是用来进行窗口按钮图案,扁平化风格的.同时,要十分注意右下角的,Border Size谁用谁知道**

我十分推荐这么几款: 

Breezemite(模仿Mac OS X的红绿灯)

Winux (Win10一模一样的窗口风格)

PlainJane(是因为我的红绿灯有问题,但是简洁好看)

### Workspace

这里不是我们配置的重点,所以大致挑一些来说.大部分维持默认即可

#### Window Management

##### Task Switcher

任务切换者,是在进行任务切换的时候.选择的方式,有多种.推荐`Flip Switch`,不过还是看个人喜好了

![Task Switcher](http://www.qiniu.evilcrow.site/KDE_TaskSwitcher.png)

#### Startup and Shutdown

##### Login Screen(SDDM)

选择登录界面,其中比较重要的就是,选择screen之后,可以自行设置Backgroud.

##### Autostart

开机自启动的软件,不多说,latte神器,这个我们后面会提到.

#### 啊啊啊啊啊啊啊 ! ! ! 找到了 !

如何设置锁屏壁纸,在`Desktop Behavior -> Screen Locking`中修改即可

#### Double-click open file

很多上手KDE国人,最不适应的一点就是: 单击打开文件,点击 + 号进行选中

那么,没有办法解决吗? 有的,见下图

`设备管理 -> 鼠标设置 -> 一般设置 -> icons,图标设置(包含打开方式)`

*KDE的基础配置差不多这么多,还有很多等待我们去发掘,如有其他有意思的设置,可以分享出来*

![Double-click to Open file](http://www.qiniu.evilcrow.site/KDE_mouse.png)

## Software

好了,重中之重,软件的配置肯定是必不可少的了.

下面会介绍在生产方面以及其他方面高效的工具了

### Vim

不多说了,"编辑器之神"不是盖得,很好用.

这里推荐一个Vim配置,超厉害,还集成了YouCompleteMe的配置,爽到.

其中目前尚未补完README,估计是个大坑,地址在此: [Crow-Vim >>](https://github.com/Evil-crow/Crow-Vim)

![Vim](http://www.qiniu.evilcrow.site/KDE_Vim.png)

### gcc套件

虽然Linux平台上,默认使用GNU C Compiler套件包括(gcc, g++, gdb等),试试clang也是不错的

尤其是clang-format,超好用,另外推荐一个二进制包集合[Binutils >>](http://www.gnu.org/software/binutils/)

### Google-chrome

新时代的超强浏览器,超爽,主要和VPN契合度也比较高.好用,而且Google 的Cloud Service特别厉害

### Visual Studio Code

虽然宇宙第一IDE--Visual Studio已经改名上了Mac OS X.不过最近还没有登录Linux平台的想法

让小弟VSC来试试水,超好用,推荐插件:

- C++ Intellisense (补全神器)
- C/C++ (提供语言支持,可以进行静态语法分析)
- One Monokai (变体Monokai,超好看)
- VSCode Great Icons (好看的Icons图标)
- Ruby Solargraph (提供Ruby补全,等着吧,Ruby劳资会回来的,不过可能上RM)

![VS Code](http://www.qiniu.evilcrow.site/KDE_vscode.png)

### Atom

说是Github出的新一代编辑器,不过目前貌似也就只是写前端舒服一点(Ruby会捡回来,前端,口可口可)

(说不定就用WebStorm去了)

![Atom](http://www.qiniu.evilcrow.site/KDE_atom.png)

### Sublime Text

使用时间很短,因为维护不积极,心累.已弃之.

![Sublime Text](http://www.qiniu.evilcrow.site/KDE_st.png)

### Qt Creator

没什么多说的,C++ Qt GUI的IDE, 写C++鸡肋一点,但是写Qt很强,图形界面做的也比较好看

![Qt Cteator](http://www.qiniu.evilcrow.site/KDE_Qt5.png)

### CLion

除VS以外的王牌C++IDE,个人审美倾向于CLion. 不过前面提到过KDE字体有问题

**曲线救国: 从Win,或者其他地方export正常的setting.jar,在这里import进行**

![CLion](http://www.qiniu.evilcrow.site/KDE_CLion.png)

### Haroopad

我见过的有史以来的最好的MarkDown编辑器,支持:

- html/css嵌套
- mathjax的支持
- 多种theme
- 语法高亮

简单地说,解释能力强,Markdown标记支持完善,同时颜值高 [Haroopad >>](http://pad.haroopress.com/user.html)

![Haroopad](http://www.qiniu.evilcrow.site/KDE_Haroopad.png)

### netease-cloud-music

超好用的网易云音乐,Fedora27目前不支持,但是github上有deb封包转化的rpm包

*有点小问题,不过无伤大雅*

![netease-cloud-music](http://www.qiniu.evilcrow.site/KDE_netease-cloud.png)

### Shadowsocks-qt5

shadowsocks的话题不多说,仅仅介绍这个好看的客户端,C++Qt编写.

好看,快捷,其中存在一个问题: 当libbortan库指定版本不存在时, 

**一般是库版本更新太快,重新做一个软链接,程序正常启动,否则会找不到链接库而启动失败**

![Shdowsocks-qt5](http://www.qiniu.evilcrow.site/KDE_ss-qt5.png)

### fcitx小企鹅输入法

fcitx只是一个框架引擎, 它可以搭载任何输入法, 我们要推荐的就是**Rime,中州韵**

具体的可以访问官网,基本是这样子. [Rime >>](https://www.baidu.com/link?url=-G25PobK8KLfBGkeuSSuWSpQmjzNqFTVj20t02l6Anu&wd=&eqid=99014f4700010abf000000035b6017de)

![Fcitx-rime](http://www.qiniu.evilcrow.site/KDE_fcitx.png)

### 文字编辑

大家可以使用Linux版的WPS,*我一般就切系统了...*

### valgrind

要想写C/C++事半功倍,必须了解（终端使用的工具）

### IntelliJ IDEA／RubyMime等

同CLion,主攻语言不同罢了,所以功能有差异,风格近似

### Latte Dock

KDE专属Dock,甩什么`dash-to-dock`, `cairo-dock`十八条街.

原生支持wayland,Fedora适性也是没问题的,超级好看兼好用

![latte](http://www.qiniu.evilcrow.site/KDE_latte.png)

---
*差不多所了,这一次的介绍就到此,我们有机会下次再补充*

*估计差不多了*

*有钱了,上Mac OS X. 当然KDE仍然是个好桌面,继续用下去*

update: July 31, 2018 4:13 PM
