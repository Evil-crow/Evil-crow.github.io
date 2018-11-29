---
layout: post
comments: true
title: 关于github绿点的常见问题--邮箱
date: May 14, 2018 4:42 PM
excerpt: 虽然最近的代码提交量少,但是还是有点干货的,但是上去一看,一直都是白的,真是让我好生苦恼,一番查询后找到了,解决方法,遂决定记录下来
categories:
- Q&A
---

*正如前面所说,今天遇到的问题就是github的绿点提交有问题,提交后并没有找到记录,很苦恼(不好装逼)*

**首先,声明本文只是解决常见的问题,便是邮箱绑定不符的问题**

若是其他的问题, 请参考: [传送门](https://help.github.com/articles/why-are-my-contributions-not-showing-up-on-my-profile/)

解决方法有以下两种:

1. 进行新的邮箱绑定 如下图:

	![](http://p8pmsq2a4.bkt.clouddn.com/git_name_email.png)

	先查看本地上进行git-push时的用户名和邮箱

	![](http://p8pmsq2a4.bkt.clouddn.com/git_email_addition.png)

	可以通过添加多个邮箱的方式来追回记录

2. 通过修改提交repo的方式来进行修改

	参考资料: [git_help](https://help.github.com/articles/changing-author-info/)

	我们可以按照下面几步来进行操作

	一, 得到需要修改的commit的repo

	```bash
	git clone --bare https://github.com/user/repo.git

	# user 为用户名
	# repo为commit的仓库名
	```

	二, 使用script文件来进行repo的修改

	sh文件内容如下:

	```bash
	#!/bin/sh
	git filter-branch --env-filter '
	OLD_EMAIL="旧的Email地址"
	CORRECT_NAME="正确的用户名"
	CORRECT_EMAIL="正确的邮件地址"
	if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
	then
		export GIT_COMMITTER_NAME="$CORRECT_NAME"
		export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
	fi
	if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
	then
		export GIT_AUTHOR_NAME="$CORRECT_NAME"
		export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
	fi
	' --tag-name-filter cat -- --branches --tags

	```

	三, 使用脚本文件修改repo,重新提交

	```bash
	cd repo.git
	bash script.sh

	git push --force --tags origin 'refs/heads/*'   # 提交新修改的repo
	```

	四, 删除旧repo

	```bash
	rm -rf repo.git
	```

	至此,便完成了追回提交记录的工作了.

	我们不得不感叹git作为一个版本控制工具的强大. git需要学习的东西还有很多.

	好(努)好(力)学(装)习(逼)

	May 14, 2018 4:42 PM
