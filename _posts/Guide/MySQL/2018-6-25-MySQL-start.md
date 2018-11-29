---
layout: post
comments: true
title: MySQL入门
date: June 25, 2018 1:42 AM
excerpt: 这两天要做数电课设,还要带电脑过去,不过你也懂我,我肯定是不会好好做的,所以慕课网还是挺有意思的,有些挺好的免费课程,这两天闲的(其实也不是闲,被闲下来...),入手一下MySQL和Docker玩玩,以后有时间了,MongDB也了解一下(Redis随缘了)
categories:
- SQL
tags:
- MySQL
toc: true
---

## 一,安装问题

此处仅仅谈到Linux平台的安装方法,Win下可以通过现成安装包方式安装

Mac没接触过,不了解.不过听说`brew`也挺好用?(反正我是`dnf`毒奶粉)

下面的流程摘自MySQL官网,相当于间接翻译一下,(...), Fc的特化版本,其他发行版同理,有区别

在官网中可以找到适合自己发行版的repo,按照说明选择的好,至于不对应版本,会有什么差错,我也不知道.

[这是官网链接>>](https://www.mysql.com/downloads/)

首先里面提供了三个版本

`MySQL Enterprise Edition `
`MySQL Cluster CGE`
`MySQL Community Edition`

如果你是个人用户,或者学习用,还是乖乖用社区版,完全足够了

[所以,这个才是真的链接>>](https://dev.mysql.com/downloads/)

左边侧栏,就可以选择合理的,安装方式,

![左侧边栏](http://p8pmsq2a4.bkt.clouddn.com/website.png)

我们以MySQL Yum Repository安装为例来说明:


### 1. 找到适合自己的repo

在下面的选择中进行repo下载:

然后使用`sudo rpm -Uvh your-system-version.rpm`

![repo](http://p8pmsq2a4.bkt.clouddn.com/repoi.png)

(隔壁的dpkg同理,packman惹不起)

### 2. 选择版本系列

习惯使用repo的人会知道,同一个repo也会有不同的版本, 而MySQL经常比较的就是5.x和现在的8.x了

*6.x是留给告诉迭代的版本的, 也就是现在5.x的后继,7.x是留给集群版本的,提前插眼*

*所以现在最新的就是8.x了,不用疑惑的*

`shell> yum repolist all | grep mysql` 查看现在支持的版本

`shell> sudo dnf config-manager --disable mysql80-community`
`shell> sudo dnf config-manager --enable mysql57-community`

进行指定版本的限制

`shell> yum repolist all | grep mysql` 重新查看,即可,要使用5.7还是8.0随意

### 3. 进行repo缓存

`shell> sudo dnf makecache`

### 4. 安装mysql

`shell> sudo yum install mysql-community-server`

会自动处理依赖的

### 5. 启动服务

`shell> sudo systemctl start mysqld` 启动服务

`shell> sudo systemctl status mysqld` 查看服务状态

### 6. 进行密码的设定

MySQL Server Initialization (as of MySQL 5.7): At the initial start up of the server, the following happens, given that the data directory of the server is empty:

The server is initialized.

An SSL certificate and key files are generated in the data directory.

The validate_password plugin is installed and enabled.

A superuser account 'root'@'localhost' is created. A password for the superuser is set and stored in the error log file. To reveal it, use the following command:

`shell> sudo grep 'temporary password' /var/log/mysqld.log`

Change the root password as soon as possible by logging in with the generated, temporary password and set a custom password for the superuser account:

`shell> mysql -uroot -p`

`mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';`

上面这段话的意思时: 启动服务,mysql已经默认创建了root@localhost用户

同时给你吧密码安排到日志文件里面了,你运行第一条命令找出密码,然后进去改密码

就这样就好了

但是,**我TM日志是空的...**

所以还有下面的邪道方法:

进入`/etc/my.cnf` 进行登录不需要密码的状态,具体是添加

`skip-grant-tables`,实现免密码登录

`mysql -uroot`  登入数据库

`update user set password=password('123456') where user='root' and host='localhost';`

修改用户名密码

记得完了修改文件,取消免密码登录,然后**重启服务**,就可以正常使用了

## 二,初始MySQL

作为一个经典的SQL,这么多年来经久不衰还是由他的理由的

有下面几个了解一下的:

1. 默认port --3306

2. 有社区版和专业版

3. 现属于Oracle公司

了解足够了

## 三,常用的最基础操作

### 1. 启动和停止服务

这个就够了,有使用Linux经验的,谁还是不会撞墙去吧...

### 2. 登录与退出

登录是这样的形式:

```bash
mysql [-option] [-arguments]
```

常用option:

1. -p --passwd[] 指定密码

2. -P 指定端口

3. -u 指定用户

4. -H 指定主机

5. --prompt 设定提示符

6. -D 指定数据库

退出就比较简单了:

```bash
shell> quit;
shell> exit;
shell> \q;
```

三种形式都可以

### 3. SQL命名规范

虽然网上说随个人习惯,不敏感大小写,但是还是遵循一下规范的好:

**命令, 函数, 保留字一律使用大写**

**数据库名, 表名, 索引名等一律使用小写**

**SQL语句一律都是 ; 结尾**

也就是说,出来是这种画风

```sql
SHOW DATABASES;
USE mysql;
CREATE DATABASE IF NOT EXISTS k1;
```

### 4. 常用操作

#### 1> 显式,切换,创建数据库

```sql
CREATE {DATABASE | SCHEMA} [IF NOT EXISTS] K2 [DEFAULT] CHARACTER SET [=] charset_name;
```

花括号为必选项, 方括号为可选项. 后面是设置编码, [IF NOT EXISTS], 可以免除一个WARNING

```sql
SHOW DATABASES;    // 显示数据库
USE database;      // 切换数据库
```

#### 2> 修改数据库属性

```sql
ALTER {DATABASE | SCHEMA} [db_name] [DEFAULT] CHARACTER SET [=] charset_name
```

比如:

```sql
ALTER DATABASE K2 CHARACTER SET utf8;   //修改K2数据库编码为utf-8
```

#### 3> 删除数据库

```sql
DROP {DATABASE | SCHEMA} [IF EXISTS] db_name
```

IF EXISTS 的用法同上

#### 4> 闲杂命令

```sql
SELECT VERSION();    // 显示版本
SELECT DATE();       // 显示日期;
SELECT USER();       // 显示用户;
```

(PS: 数据库操作还是挺好玩的,我都是跑路了,当个数据库管理员也不错,开玩笑的,还是写C/C++舒服)

***
