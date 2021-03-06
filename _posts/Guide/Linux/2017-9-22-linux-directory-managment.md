---
layout: post
comments: true
title: 第九篇 Linux文件与目录管理(下)
date: September 22, 2017 4:47 PM
excerpt: 上一篇提到了LInux系统下文件/目录的基本操作,新建,移动,拷贝,删除.以及对于文件的各种形式的查看那么这一篇所要谈到的就是,文件/目录的隐藏属性以及文件/目录的查询操作
categories:
- Unix/Linux
tags:
- vbird
---

*上一篇提到了LInux系统下文件/目录的基本操作,诸如:切换目录,*

*对于目录/文件的新建,移动,拷贝,删除.*

*以及对于文件的各种形式的查看*

*那么这一篇所要谈到的就是,文件/目录的隐藏属性以及文件/目录的查询操作*

_ _ _

### 1.文件/目录的默认权限和和隐藏权限

进行了之前的了解,你应该已经自己动手创建过文件或者目录了吧?

但是你有没有好奇过,每次新建文件/目录,他们的出生权限都是一致的.
![](http://img.blog.csdn.net/20170617190716652?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
如上图所示,新建test1,test2,他们的权限都是664

这就要牵扯到文件/目录的**默认权限**与**隐藏权限**了

_ _ _

#### 1.1 文件默认权限:umask

**默认权限:当前用户在新建文件/目录时,文件/目录的默认权限**

使用umask命令即可查看当前默认权限

```[Evilcrow@Evilcrow]$ umask```

或者

```[Evilcrow@Evilcrow]$ umask -S```

上图中的0002即为当前的默认权限

看见这个0002你是不是也是一脸懵比?,没错,一开始我和你也差不多.

对于0002默认权限的理解:

首先需要明确的是,在新建文件/目录时,对于文件/目录的权限分配:

> 对于文件,默认是没有X,可执行权限的,因为不确定文件是否可以执行

> 即文件 -rw-rw-rw-,所以文件权限默认最大666

> 对于目录,默认是权限全开,即目录 drwxrwxrwx,所以目录的默认权限最大为777

接下来,再来理解umask,默认权限的数字0002,其中后三位,分别为user,group.others的rwx权限

**所谓"默认权限"的意思是:在最大权限上减去默认的权限,即为目前的默认权限**

举个例子:umask 0002

对于文件:**-rw-rw-rw-减去 --------w-,则文件的默认权限为-rw-rw-r--,即664**

对于目录:**-rwxrwxrwx减去 --------w-,则目录的默认权限为-rwxrwxr-x,即775**

到这里,你就了解了umask以及它与默认权限的关系了吧!

既然默认权限影响着我们进行工作配置时的快捷与否,那么该如何进行默认权限的修改呢?

```[Evilcrow@Evilcrow]$ umask number```

在**umask+数字**即可修改默认权限.

**注:在未进行修改的时候,root用户的默认权限是022,一般用户的默认权限是002**

_ _ _

#### 1.2 文件的隐藏属性 lsattr,chattr命令

在之前的内容中,你可能也曾注意到,文件的权限中有10个字符,9个为rwx权限,

那么剩下的一个是什么呢?

**剩下的一个,即为"隐藏属性"**

隐藏属性,可以称之为文件的高级属性,隐藏属性有很多,其中有几个最常用的

##### chattr 命令

**chattr 用来进行隐藏属性的更改(change attr)**

```[Evilcrow@Evilcrow]$ chattr [-+=] [ASacdisu]  文件/目录```

chattr命令,也类似于chmod,使用-+=来赋予文件/目录以隐藏属性

> -A 访问文件时,atime不会发生改变(access time),可以避免过度访问磁盘,保护磁盘

> -S 一般文件是"异步"写入磁盘的,关机前才会有sync命令,使数据写入磁盘,-S可以"同步"写入磁盘

> -a (重要)设置此权限后,文件只能增加数据,不能修改数据,也不能删除数据,只有root才有此权限

> -c 此参数使得文件存储时自动压缩,读取时自动解压,方便存储大文件

> -d dump程序运行时,不会使文件被dump程序备份

> -i (重要),-i可以使文件"不能被改名,删除,设置连接也无法写入或者添加数据"只有root有此权限

> -s 设置s属性,当文件被删除时,完全被删除,并不会进行备份,(相当于shift+del)

> -u 与-s相反,删除文件后并没有被完全删除,可以通过一定方法找回数据

以上即为chattr可以修改的隐藏属性,也是文件的所有隐藏属性

**注:其中的-a,-i参数十分重要,对系统的安全也有极大的保护,但只有root才有此权限**

chattr用来修改文件属性,那么修改后的文件属性如何查看?

**使用lsattr命令**

```[Evilcrow@Evilcrow]$ lsattr [-adR]  文件/目录```

> -a 将隐藏的文件属性也全部显示出来

> -d 如果接的是目录,则不显示其中的文件,而只是目录本身的隐藏属性

> -R 连同目录下的子文件隐藏属性也全部显示出来

使用chattr命令修改隐藏属性,lsattr命令查看隐藏属性,系统管理效率事半功倍

_ _ _

#### 1.3 文件的特殊权限: SUID,SGID,SBIT

当时现在umask查看默认权限的时候,你可能看到有四个数字,既然后面的是u,g,o的rwx权限

那么,第一个是什么呢? 就是文件的特殊权限

之前在目录树中也有几个特殊的目录,仔细观察
![](http://img.blog.csdn.net/20170617191118232?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![](http://img.blog.csdn.net/20170617190835263?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
上图中passwd的权限为-rwsr-xr-x,这个S即为"特殊权限"

我们来考虑一个问题,如果对于某一类文件,他的权限属于root用户

权限为"-r--------"

但是一般用户又必须具有W的权限才能进行文件操作,

这个时候特殊的文件权限就派上用场了

有这种例子吗?

**最常用的: 密码**

一般用户明明不具有密码文件的W"写"权限,但是它却可以修改自己的密码,这就是特殊权限

**实质上,特殊权限,相当于对于非权限内用户,特殊的执行权**

特殊权限有三种:SUID,SGID,SBIT

_ _ _

#### SUID 特殊权限

**当s出现在user上时,称为Set UID,即SUID**

> SUID权限仅对二进制程序有效,(即对shell script无效)

> 执行者对于此程序有X权限

> 这个特殊的执行权限仅仅在执行过程中生效

> 执行者将具有程序拥有者的权限

上面的话很生硬对吗?

我用自己的话帮大家总结:

**权限外用户,(对此文件有x权限),在执行程序期间,获得程序所有者权限,仅对二进制程序生效**

举个实际的例子:

**密码文件属于root,但是一般用户在使用期间,却可以获得密码文件的使用权,以及修改权(仅限自己的)**

_ _ _

#### SGID 特殊权限

**当s出现在group中时,称为 Set GID,即SGID**

如上图

**SGID不同于SUID权限,SGID可以使用在目录和文件上,但是SUID只能使用在二进制程序上**

对于文件而言:

> SGID对二进制程序有效

> 程序执行者对文件需要具有x权限

> 执行者在执行过程中将获得用户组的支持

对于目录而言:

> 用户需要对于该目录具有r与x权限,才能进入该目录

> 用户在此目录下的有效用户组,将会变成该目录的用户组

> 若用户对于改目录具有w"写"权限,那么新建的文件用户组与目录的用户组相同,而非用户的用户组

对于SGID在目录上的应用,我总结一下:

**当权限外用户,具有r,x权限(对目录)后,可获得目录用户组的支持,而且,W权限的操作,为目录用户组所为**

**SGID这个特殊权限对于文件的作用不谈,但是对于目录的特殊权限太重要了**

**尤其是在,进行项目目录的构建时,设置SGID权限,则项目成员所完成的项目文件同属一个用户组,方便管理**

_ _ _

#### SBIT 特殊权限

**这个权限目前只对目录有效,当"t"出现在others中时,即为SBIT**

> 当用户对此目录具有w,x权限时,该用户便具有了写入的权限

> 当用户在该目录下创建文件或者目录时,只有自己和root用户才有权限进行删除操作

那么，SBIT特殊权限意义何在呢？

**同上理，在进行项目组的开发时，同一用户组的成员进行协作，在同一目录下进行工作**

**任何用户都可以再次目录里进行新建和修改文件，但是只有文件创建者和root才有权限进行删除**

**这样，是不是很方便呢？**

##### 特殊权限的修改

看过上面的内容,你应该了解特殊权限的巨大作用了吧?

那么,特殊权限如何修改

```[Evilcrow@Evilcrow]$ chmod abcd 文件```  abcd为文件权限的权值

之前的权限修改是 chmod+xyz,那么现在再加上一个数字用于特殊权限的修改不就好了

权值: SUID--4 , SGID--2 , SBIT--1
![](http://img.blog.csdn.net/20170617190947715?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
如上图所示,给目录添加了SBIT权限,SUID,SGID权限的修改同理

**有没有发现,第一次修改后,test目录的SBIT权限为"T"?**

> 这是因为,test的权限为1644,它没有可执行权限x,当修改特殊权限SBIT时,

> 没有可执行权限,t也就不复存在了,所以"T"含义是"空"

> SUID,SGID中的"S"同理,表示空

当然进行特殊权限的修改时,也可以使用符号

```[Evilcrow@Evilcrow]$ chmod u+s/g+s/o+t  文件```

_ _ _

#### 查看文件类型

对于一个文件,我们可以使用```ls -l 文件```来显示他的所有属性

但是,如果只是要知道文件类型,使用file命令更方便呢!

```[EVilcrow@Evilcrow]$ file  文件/目录```

![](http://img.blog.csdn.net/20170617191046282?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
如上图所示,file命令可以清楚的知道每一个文件/目录的类型

### 2. 命令与目录的查询

*前面提到的都是文件/目录的操作,查阅,修改权限.你首先要能做到的是查阅文件*

#### 2.1 脚本文件明的查询

#### which 命令

用于查找脚本文件(简单来讲,查找名令)

```[Evilcrow@evilcrow]$ which [-a] command```

which用于查找命令所在目录位置,当然这就扯到之前的PATH,which是在PATH中查找命令

所以有时候需要进行更新内容才能显示出来,

> -a ,次参数,可以将command在PATH中的所有路径显示出来,并非只是最先找到路径

**注:既然说了是在PATH中寻找路径,那么,不再路径中的目录肯定就是找不到的了!**

**别惊讶,这样的command可是不少的呢!**

_ _ _

#### whereis与locate 命令

上面的which是脚本命令的查找,那么接下来就是文件的查找了

进行文件的查找,所需要的命令是whereis与locate命令

```[Evilcrow@Evilcrow]$ whereis [-bmsu] 文件/目录名```

> -b 表示仅查找二进制的文件

> -m 仅查找再说明文件manual下的文件

> -s 仅查找source源文件

> -u 查找其他类型的文件

```[Eilcrow@Evilcrow]$ locate [-ir] keyword```

> -i 忽略大小写的差异

> -r后面可接正则表达式的形式

以上即为whereis与locate的用法

**但是,请注意,whereis与locate查找的依据是各自的数据库!!!**

**意即,有时你会查找到删除了的文件,有时也会查找不到新建的文件,这就**

**需要更新数据库了 updatedb**

```updatedb```:根据/etc/updatedb.conf的设置进行数据库内容的更新

```locate```:依据/var/lib/mlocate内的数据记载,找出用户输入的关键字文件名

**让我来理解,即wheris与locate的搜索很快,但是需要更新数据库**

接下来,介绍find命令,

_ _ _

#### find 命令

上面说的whereis与locate命令并非是在硬盘中查找数据,而是在各自得数据库中查找

**然而,find命令,就是在硬盘中进行文件查找的命令**

```[Evilcrow@Evilcrow]$ find [PATH] [option] 文件/目录名``` find命令

这里为什么不再格式里写上find命令的参数呢?

**因为find,功能强大,它的参数太多了 !**

> 1.与时间有关的参数(atime,stime,mtime),下面以mtime进行说明

> -mtime n:n为数字,表示在n天之前的那个"一天之内"所修改过时间的文件名

> -mtime -n:列出n天之内(包括n天在内),修改过文件时间的文件名

> -mtime +n:列出n天以上(不含n天在内),修改过文件时间按的文件名

**注:n = 0表示目前的时间**

**其中,请切身体会,n,+n,-n三种不同时间限定所带来的文件搜索不同,**

**并且,时间搜索,还有atime与stime,同mtime理**

> 2.与用户和用户组有关的参数

> -uid n: 第四章详解

> -gid n: 同上

> -user name:  搜索特定目录下,name用户的文件

> -group name:  搜索特定目录下,name用户的文件

> -nouser   :寻找文件的所有者不在,系统文件中记录的内容

> -nogroup  :同上(用户组)

**最后的两个参数很适合去查找一些不太正常的文件或者目录出来**

> 3.与权限有关的参数

> -name filename :按照文件名查找文件

> -size [+-] SIZE:查找比SIZE大(+)小(-)的文件,SIZE的单位有,B,K

> -type TYPE :查找文件类型为TYPE的文件,-,d,l,s,p等

> -perm mode:查找文件权限恰好为mode的文件

> -perm -mode:"查找文件权限比需要全部包含mode的文件"

> 例如:-perm -755,则777符合条件

> -perm +mode:"查找文件权限包含任一mode权限的文件"

> 例如:perm +755,则701也符合要求

**事前谈到过的,特殊权限SUID,SGID,SBIT,同样也是可以用在这里的,比如:4755**

> 4.一些特别的参数:

> -exec command: command为其他的命令,-exec,接其他的命令用来处理查找的结果

> -print   ：将处理的结果打印到屏幕上

举个例子：

```[Evilcrow@Evilcrow]$ find / -perm +7000 -exec ls -l { } \;```则

这条命令表示在根目录下查找权限含任一7000的文件，并将它的文件信息ls -l显示出来


**上例中，{}表示查找到的内容，－exec一直到＼都是关键字，在其中间的就是额外命令**

**而且，重要的是，可以使用通配符进行查找操作，十分方便，同时，多重参数互相组合，更方便**

**最后一点，不到最后时刻，find命令真的很费时，因为它是直接在硬盘中进行搜索**
_ _ _

### 3.权限与命令间的关系

*此处的内容十分关键，请认真对待*

> 使用户拥有进入目录的权限，使用CD命令

**至少需要对该目录有ｘ权限，如果需要读取目录中的文件，则必须要有ｒ的权限**

> 使用户具有在目录内读取文件的权限，可使用cat,more,less命令

**如果是读取目录，至少需要ｘ权限，对于文件，需要ｒ权限**

> 使用户具有修改一个文件的权限vi,vim,nano,emacs

**如果是目录，需要ｘ权限，如果是文件，至少需要ｒ，ｗ权限**

> 使用户具有创建文件的权限，如mkdir,touch

**对于目录需要有ｗ，ｘ权限，重要的是ｗ权限，因为没有ｘ权限，无法进入目录**

> 使用户具有进入一个目录并执行目录下文件的权限

**对目录，至少要有ｘ权限，对于文件至少要有ｘ权限，已知文件名的情况下，可以没有ｒ（目录）权限**

**以后内容十分重要，需认真进行分析理解**

_ _ _

*这一篇的内容也到此为止了，到现在Linux系统中的文件/目录从配置到操作，到分析查阅就结束了*

*从下一篇开始，就是磁盘与文件系统的管理了，下次再见　！*

06/17/2017

