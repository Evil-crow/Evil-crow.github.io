---
layout: post
comments: true
title: 第十篇 磁盘与文件系统管理
date: September 22, 2017 4:56 PM
excerpt: 正如标题所言,磁盘与文件管理系统的内容,下面就听我一一道来吧
categories:
- Unix/Linux
tags:
- vbird
---

*距离上一篇blog也有几天的时间了,一是偷懒了,二是这一篇的内容确实挺麻烦的..*

**那么,这一章的主要内容是什么呢?正如标题所言,磁盘与文件管理系统的内容,下面就听我一一道来吧**

##### 一.相关的硬件知识

|磁盘的组成|               作用
|---------|
|磁盘盘面  |              记录数据的地方
|机械手臂与磁盘头|         用来读写磁盘内容的工具
|主轴马达|                使机械手臂转动

既然盘面是其中最主要的部分,那就来谈谈盘面吧!

**盘面 = 扇区 + 磁柱(由扇区组成一个圆)**

**其中,第一个扇区最后重要:boot file (446 Bytes) + 分区表 (64 Bytes)**

**关于SATA与IDE接口的磁盘:SATA为/dev/sd\*   IDE为/dev/hd\***

注:分区表为64bytes 所以,只能写入4组分区信息(主分区 + 扩展分区[逻辑分区] <= 4)

磁盘 = 硬盘 + 软盘

_ _ _

##### 二,文件系统(filesystem)的知识

为什么会有文件系统这一说,不是所有的都是在硬盘上的吗?

**最早,因为硬件及技术的原因,并不能使多个文件系统在一块硬盘上共存**

**但是,随着技术的发展,LVM以及磁盘矩阵等一些新技术的发展,使得同一块磁盘系统,可以有多种文件系统**

**,意即,可以对同一块磁盘进行分区,使其上不同的分区有着不同的文件系统**

谈及文件系统,自然就要说说文件系统上到底保存了哪些数据?

|  数据类型    |   作用 |
|-------------|-------|
|SuperBlock|用来记录文件系统的整体信息,block/inode的管理信息|
| inode | 记录文件的属性和权限,其中存储着对应的Block号码|
|  Block |记录文件的实际数据,文件数据过大时,会占用多个Block|

**文件系统中,使用以上的数据进行构建的便是"索引式文件系统"**

**整体思路是,inode存储文件权限和属性意即对应的Block号码**

**而实际的文件数据存储在Block中,通过inode进行Block的索引查找**

**不同于,索引式文件系统,闪存盘里所使用的FAT文件格式(包括Win平台)则没有inode的设定**

**仅使用Block进行文件数据的存储,同时Block记录着下一个Block的号码**

相比之下,**索引式文件系统的工作效率和可维护性更高**

原因在于:

> 索引式文件系统,block的号码都存在inode,可以一次性的把文件的Block号码全部获取,进而访问,快,高效

> 相对的,FAT文件格式的文件系统,效率低,一次只能读取一个Block,

注:**关于磁盘整理的问题,FAT中,因为不断进行数据的读写,擦除,会使得同一个文件的Block零散化,**

**最终的访问效率,不忍直视,所以有磁盘碎片整理的文件功能,但是,对于索引式文件系统,这个问题就不是很明显**

**一般情况下,不需要对inode进行整理,除了特别的例外情况以外,Block号码过于复杂,也是需要进行整理的**

**一般情况下,索引式文件系统是不需要进行磁盘碎片整理的**

既然我们学习,使用Linux,那就重点看一看linux下的文件系统把!

Linux下最早使用的便是Ext2文件系统(**现在最新的已经是Ext4了!**)

而Ext2文件系统就是基于,索引式文件系统所建立的.

接下来,详解Ext2文件系统:

**注:因为特别大的文件磁盘里,inode与block是要进行分区的,不然会混乱的**

所以,在Ext2文件系统里,划分出了多个blockgroup供用户使用

每个blockgroup中所包含的内容有:

**SuperBlock + Inode table + Datablock + Block bitmap + Inode bitmap + filesystem description**
_ _ _
##### Data Block:数据块,实际存储数据的地方

**每个Block都存在编号,而且,每个Block的大小在文件系统建立时就已经确定 1K,2K,4K**

|  条目   | Block的限制|
|-----|-----------|
|1.   |原则上,Block的大小和数目在文件系统建立后便不能再发生变动|
|2.   |一个Block中只能存储一个文件|
|3.   |接第二条,当存储不下时,会多分配Block|
|4.   |接第三条,当存储空间有剩余时,保留空间,不能存储其他文件|

因为以上限制,所以,在进行block得分配时,需谨慎选择

大:造成存储空间的浪费

小:会造成过多的inode号码,降低读写效率
_ _ _

##### Inode Table:inode 记录区域

inode中存储的是文件的权限和属性,

Inode Table中存储的内容:

**权限 + 属性 + 容量 + ctime + atime + mtime + SetUID + 真正的Block指向**

| 条目   |inode的限制|
|----|----------|
|1.  |每个inode站固定大小128bytes|
|2.  |每个文件会占用一个inode|
|3.  |filsystem能建立的文件数与inode有关|
|4.  |读取时,先找到,inode,权限符合后,才能进行Block号码的读取|

因为 inode仅仅128bytes,并不能存储过多的Block号码,但有的时候又需要特别多的Block号码进行存储

怎么办呢?大佬们,已经解决这个问题了

**12 + 1 + 1 + 1,12直接,1间接,1二重间接,1三重间接进行inode号码的存储,即进行多次间接指向**

_ _ _

##### SuperBlock:用于存储文件系统的相关信息

上面提到的inode,以及Block的内容,的相关信息,都是存储在SuperBlock当中的

SuperBlock中存储的相关信息:

**Block,inode的相关用量 + filesystem的挂载,最近一次写入文件的时间,valid bit的数值**

SuperBlock中一般为1KB的信息

**在一个文件系统中,一般第一个blockgruop中为整个filesystem中的superblock信息,而其他的blockgroup中的superblock信息都是其备份**

_ _ _
##### Filesystem Description :文件系统的描述说明

用于查看每个blockgroup,从开始到结束的号码,以及blockgroup中每个区段分别介于的号码

##### Block/Inode bitmap:区块/索引节点(Block/Inode]对照表

用来保存哪些Block(inode)非空,哪些为空的标记信息,在进行文件的读写时十分有用!
_ _ _

#### 三,文件系统与目录树的关联

上面的便是文件系统(filesystem)的基本内容,那么这个东西如何与目录树有关联的呢?

**一个目录,实际上也是一个文件,其中filesystem会给目录分配一个inode及一个(多个)block**

**目录中所记录的是,文件名及其对应的inode号码**

藉此进行文件的访问,也正式因为Ext系列文件系统的索引式结构,所以,才能有接下来软,硬连接两种方式

_ _ _
此处,进行ls,知识的拓宽:(毕竟要自己实现一个ls)

1. ls -i 用于显示inode号码的命令

2. ll后面的一行信息中数字4096,之类的表示目录的容量,连接数的含义:有多少个文件连接到这个inode号码上

**我来解释一下,为什么时4096这些数字,这与block的大小有关,都是默认大小4K的整倍数!**

**问题又来了,为什么有的就不是4K的整倍数,因为,有的问就不是同一文件系统!使用```dumpe2fs```查看**

例如/proc,swap,更甚,这些只是虚拟内存中东西,并不占实际的内存空间

**之前说过Ext2/3/4 的filesystem基本不需要进行磁盘碎片整理,但如果非要整理该怎么办呢?**

**正解:将所有文件拷贝出来,之后重新格式化磁盘,最后将文件重新写入!**
_ _ _

继续上面的内容,

在文件系统中,进行文件的存取时的步骤:

- 用户对该目录是否具有w,x权限,目录的inode

- inode bitmap,找到为使用的inode,写入为文件的权限和属性

- block bitmap 找到未使用的block,写入数据,并更新inode中的,block指向

- block bitmap,inode bitmap.superblock的数据更新

_ _ _

#### 四,日志式文件系统的出现

众所周知,在使用电脑时,难免会出现一些意外情况,断电,死机,那么数据怎么办?

肯定是开机后进行修复排查,但是,太过耗时,耗力,很费时间.

所以,前辈们便开发出了,**Ext3日志式文件系统**

**其实现原理是,在filesystem在重新划出一个区块来,用来记录写入/修改的信息**

**万一出现意外情况,那么,只检查journal中的记录日志,便可以迅速找出目标区块,大大提高了效率**

**另外,可以通过异步处理sync,来解决这个问题.**

**实际上,异步处理就是系统不定时的把编辑过的文件写入磁盘中,可以用sync强制写入**
_ _ _

#### 五,关于Linux文件系统中的连接问题(软/硬连接)

#####  1.硬连接(Hard-Link)

硬连接,需要与之前见到过的快捷方式(软连接有所区别)

硬连接(Hard Linuk):

在Ext4 filesystem中,这一类的索引式文件系统中,inode与block的关系正如上文所讲

即,查找,访问一个文件,关键的是,先找到文件inode,便可以访问到文件

所以,就有了这样的一种连接:硬连接

**在目录信息中(block)所写入一条数据/关联数据,用来连接到目标文件的inode上**

**这样下来,创建的硬连接,实际上便是源文件的别名,而文件的inode上便连接到了两个文件名与其关联**

**所以,ll之后,文件的连接数都会加'1',这两个文件名,是除了文件名(路径)之后,属性完全相同的数据**

而且,因为硬连接,只是在其中block中,多写入了数据,并未分配inode,所以不能算是新建文件

如下图

![](http://img.blog.csdn.net/20170719153607870?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

硬连接的优劣:

**优: 硬连接因为不创建新的文件,所以相比较于软连接,不会占用大量的内存空间,**

**同时,硬连接可以理解为给文件起别名,完全一致,所以,误删其中一个文件,不会影响文件的打开情况**

**劣**

**1.硬连接不能跨文件系统,因为不同的文件系统中inode可能会有重复,胡乱连接,会出错的**

**2.硬连接不能连接目录,因为连接目录意味着,还要与目录下的每个文件建立连接,不仅任务量大,而且目录内容的更新,也会影响着硬连接内容的更新**

##### 2.软连接(Symbolic-Link)

理解了上面的硬连接,下面软连接就比较好理解了

访问文件是要去寻找文件的inode号码,而文件保存在目录中,目录的block保存着文件的inode信息

**硬连接是连接文件在目录block中储存的inode号码**

**而软连接,则是去连接文件名的inode,并非是与文件名对应的,储存实际数据的inode号码**

**可以理解为,直接连接与间接连接**

如下图:

![](http://img.blog.csdn.net/20170719153624176?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

软连接的优劣:

**优:跨目录,跨文件系统,能力十分强大**

**劣:在删除源文件后,便不能访问文件,基本上可以近似理解软连接为Win下的"快捷方式"**

![](http://img.blog.csdn.net/20170719153638233?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvRV92aWxjcm93/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

而且,注意:软连接,实际上是分配了inode与block,十一个新建的文件

_ _ _

#### 六,关于磁盘的操作知识

*先说说我自己学习这部分的感受,这部分知识,学得快,忘得快,这部分的知识,其实更主要的是命令的操作,*

*是要你能够熟练得运用名命令,便可以很好的处理这些磁盘问题*

*但是,我自己是不喜欢这部分知识的,虽然很重要,但是,枯燥,乏味,而且好多东西用不到*

*比如我自己挂载U盘时,从来都不给我机会,要不自动挂载,要不就,不挂载*

*而且,也听了Hg_Yi的建议,上面中重要的知识,需要进行详细的记录,但是这些命令的操作,仅提供思路,功能*

*实际上去man或者info命令,比我自己记录要来的更好*

**磁盘分区:fdisk,root权限,fdisk更像是一个程序,根据其中的help要求操作即可**

**磁盘格式化:mkfs 即可设置文件系统,然后格式化对应的磁盘 , mke2fs提供更详细的设置**

**磁盘检验:fsck 当文件系统遭遇一些可怕的情况时(停电,死机)时,用此命令进行检验修复**

**磁盘的挂载与卸载:完全靠mount这个强大的命令,其详细使用方法,man mount**

**磁盘参数的修改:mknod,修改磁盘的对应参数,使其成为指定的设备文件**

**开机挂载:便是将挂载信息,写入指定文件中 /etc/fstab 与 /etc/mtab**

**特殊的loop挂载:..**

**Swap分区的构建:其中包括了DD建立大文件 + loop挂载两部分内容**

_ _ _

#### 七,文件系统中得一些小问题

##### 1.boot sector 与 super block的关系

这个要涉及block的大小

**若为1K则,boot扇区在前,superblock在后,各占1K空间**

**若为2K或者4K,则boot扇区内容占了0-1024号内容,superblock占了第二部分的内容**

**之后,这个block的内容保留,不再存储其他文件数据**

##### 2.磁盘空间的查看

du -s 与 du -S不同

-s会记录子文件夹,所以,最后空间并不会十分准确

##### 3.parted进行分区

当单磁盘的空间达到2TB时,不适合继续用fdisk进行后续的处理,应该使用parted进行处理

_ _ _

本篇的内容,到此告一段落,最近理完了系统操作中最复杂的磁盘与文件系统,就需要进行Linux_C的学习了

下次会进行Linux中,十分有用的打包与压缩命令的学习

**另外 @Hg_Yi,写Blog抓住重点很有用,但是在进行系统的学习时,我还是继续详尽的进行记录**

**等到之后的编程学习时,便可以有效的进行总结**

July 19, 2017 3:32 PM
