---
layout: post
title: GDB调试系列(一)
date: August 8, 2018 2:34 PM
excerpt: 众所周知,GDB是一个强大的调试工具.这些没用的吹逼话就不多说了.GDB好不好用,谁用了谁知道,不过好的Bug真的是可遇不可求.每一个Bug都是上辈子的缘分, 呸, 胡侃罢了..
categories:
- Unix/Linux
tags:
- GDB
comments: true
---

*摘要里面也说了,既然GDB功能如此强大,毋庸置疑.它的学习曲线也就不是一般的麻烦*

*这里我们暂时不去追究它的细节,而是就事论事,对于这种工具,需求驱动的效率会更高一点.*

*那么,就近入主题吧,Mission Start ~*

~~起因是学妹在写myshell的练习时,遇到了奇怪的问题, 我都忘的差不多了...~~

问题: **管道命令,第一次输出无效,之后的输出全部有效**

而且此前已经查实,参数的解析是没有问题的.这种束手无策的时候,不用多想, GDB上去怼.

[源代码 >>](https://github.com/XiyouLinuxGroup-2018-Summer/TeamD/blob/master/Code/%E8%B5%96%E9%91%AB/%E6%97%A5%E5%B8%B8%E7%BB%83%E4%B9%A0/%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6/6.c)

恐怕以后链接失效, 在这里贴出要重点调试的代码...

```cpp
case 4:
	if(pid==0)
	{
		pid_t pid2;
		int fd2;

		if((pid2=fork())<0)
		{
			printf("\033[31m 进程２创建错误\n\033[0m");
			return;
		}
		if(pid2==0)
		{
			if((find(arg[0]))==0)
			{
				printf("\033[31m %s：找不到命令\n\033[0m",arg[0]);
				exit(0);
			}
			fd2=open("/tmp/newfile",O_WRONLY|O_CREAT|O_TRUNC,0644);
			dup2(fd2,1);
			execvp(arg[0],arg);
			exit(0);
		}
		waitpid(pid2,NULL,0);

		if((find(argnext[0]))==0)
		{
			printf("\033[31m %s：找不到子命令\n\033[0m",argnext[0]);
			exit(0);
		}
		fd2=open("/tmp/newfile",O_RDONLY);
		dup2(fd2,0);
		execvp(argnext[0],argnext);
		if(remove("/tmp/newfile"))
		{
			printf("remove error\n");
		}
		exit(0);
	}
	break;
```

*这个代码与我们平时写的玩具有不同之处在于, 它是multprocess的(多进程的)*

**那么,我们就要注意如何使用GDB调试多进程的程序了**

> GDB在调试多进程符方面还是又不错的性能的,(多线程就不表了)

> 其中涉及的主要参数是:

> follow-fork-mode , 表示追踪进程的模式,有parent, child可选

> detach-on-fork , 表示是否分离子进程, 有off, on可选

> info inferiors , 查看目前的进程表

> inferiors ID , 可以切换到对应进程,(注意: 是GDB划分的ID, 并非进程的PID)

基本流程:

- 设置追踪模式(子进程/父进程), 分离方式(分离/不分离)

- 调试程序 { gdb execulate }

- 标记断点 { b | breakpoint } line

- 运行程序 { r / run }

- 切换进程 { inferiors ID }

如下图:

我们终于找到了Bug所在.

![设置属性](http://p8pmsq2a4.bkt.clouddn.com/gdb_multprocess1.jpg)

![切换进程](http://p8pmsq2a4.bkt.clouddn.com/gdb_multprocess2.jpg)

![收到错误信号](http://p8pmsq2a4.bkt.clouddn.com/gdb_multprocess3.jpg)

*那么,问题找到了,仅下来的Bug就好好改吧*

**WARNING: 恰当的例子,真的可遇不可求, 我们应该用他们好好练手** 

**如果大家有不错的例子,也可以私戳我, 啊, 你说啥, 线程的调试?等学妹这一周的Bug吧**

