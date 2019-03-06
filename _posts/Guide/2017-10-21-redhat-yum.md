---
layout: post
comments: true
title: 关于RedHat本地源的配置
date: October 11, 2017 7:14 PM
excerpt: RedHat是很多人青睐的Linux发行版,一方面是因为他在企业级服务器方面使用的最多,另一方面则是其对整个业界作出了很大贡献,有诗云:"我为社区做贡献,社区让我赚大钱",说的便是RedHat是也
categories:
- Guide
---

*而摆在RedHat用户面前的第一个问题,就是关于本地源配置的问题*

*因为,RedHat是主打服务的.所以,RedHat软件源列表的更新是付费的.*

## 配置原理

如何修改RedHat的包管理器呢?

我们的思路是:更换掉RedHat本身的包管理器,而使用与其更加相近的CentOs的包管理器进行配置

## 配置组件

因为一些版本的兼容的问题,我在这里将一套可以完整使用的rpm名称贴在这里

- python-urlgrabber-3.10-8.el7.noarch.rpm

- yum-3.4.3-132.el7.centos.0.1.noarch.rpm

- yum-metadata-parser-1.1.4-10.el7.x86_64.rpm

- yum-plugin-fastestmirror-1.1.31-34.el7.noarch.rpm

- yum-updateonboot-1.1.31-34.el7.noarch.rpm

- yum-utils-1.1.31-34.el7.noarch.rpm

而这些包去哪里找呢?

到 [RPMsearch](http://rpm.pbone.net) 去寻找

## 配置步骤

第一步 : 查询所安装的RPM包(yum)

```bash
rpm -qa | grep yum
```
![](http://www.qiniu.evilcrow.site/Guide_rpm-qa.png)

第二步 : 解除依赖,并卸载所有的自带yum包

```bash
rpm -aq | grep yum | xargs rpm -e --nodeps
```

第三步 : 进行rpm包的安装

```bash
rpm -ivh python-urlgrabber-3.10-8.el7.noarch.rpm
rpm -ivh yum*               (注意这几个包因为相互依赖,所以同时安装可以取消问题)
```

第四步 : 配置新的CentOs源

```bash
wget http://mirrors.163.com/centos/6/os/.......(自行查找)
```

之后,将下载到的repo文件移动到/etc/yum.repos.d目录下

使用Vim打开进行修改

```bash
:%s/$releasever/7/gc
```

第五步 : 进行本地元数据的更新

```bash
yum clean all
yum makecache
```
**到此为止,新配置的yum及源基本可以使用了,之后进行软件的管理,只要继续添加,修改源就可以了**

![](http://www.qiniu.evilcrow.site/Guide_mysql.png)

## 相关声明

**首先,RedHat的确是一个为了社区及发行版的发展而不懈努力的公司**

**配置RedHat本地源,并没有进行商业利益损害的想法,只是进行基础的技术交流而已**

**而且,这样配置好的包管理器,在依赖上的打磨没有RedHat处理的好,**

**如果喜欢RedHat,建议进行用户的注册.同时CentOS以及Fedora也是不错的发行版**

October 11, 2017 8:09 PM
