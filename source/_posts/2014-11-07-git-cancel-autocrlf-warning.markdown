---
layout: post
title: "Git取消LF will be replaced by CRLF警告"
date: 2014-11-07 09:18:42 +0800
comments: true
tags: Git
keywords: git, LF, CRLF
description: Git取消LF will be replaced by CRLF警告
---
最近使用Octopress进行`rake deploy`时，控制台一直提示一堆如下警告信息：

	The file will have its original line endings in your working directory. warning: LF will be replaced by CRLF in Gemfile.
	The file will have its original line endings in your working directory. warning: LF will be replaced by CRLF in Gemfile.lock.
	The file will have its original line endings in your working directory. warning: LF will be replaced by CRLF in README.

每次提交都要面对如此多的警告，确实忧伤。<!-- more -->好吧，赶紧求助谷歌爸爸。[Stackoverflow]()真是个好地方，不仅解释了为什么会有类似警告，还提供了解决的办法，这可谓VIP一条龙服务啊。  

> In Unix systems the end of a line is represented with a line feed (LF). In windows a line is represented with a carriage return (CR) and a line feed (LF) thus (CRLF). when you get code from git that was uploaded from a unix system they will only have a LF. It's nothing to worry about.

原来是因为Windows和Linux在处理文本换行的差异引起的，在Windows中一行的结束符由carriage return(CR) + line feed(LF)组成，也就是CRLF；在Linux中没有CR，只有LF。大家都知道Git和Linux都是[Linus Torvalds](http://en.wikipedia.org/wiki/Linus_Torvalds)写的，理所当然Git服务端也运行在Linux系统上，当我们提交本地文件到远程仓库，Windows下的换行规则就要替换成Linux下的换行规则，也就有了上述警告。  
所以呢，这类警告是完全不会影响Git的正常使用。但是，所谓眼不见心不烦啊，能不能让Git不显示这类警告呢。答案是，当然可以，而且方法也很简单，改下Git的配置文件即可。  
打开Git Bash。

	$ vi /etc/gitconfig

在`[core]`下将`autocrlf = true`更改为`autocrlf = false`。

	[core]
	        symlinks = false
	        autocrlf = false
	[color]
	        diff = auto
	        status = auto
	        branch = auto
	        interactive = true
	[pack]
	        packSizeLimit = 2g
	[help]
	        format = html
	[http]
	        sslCAinfo = /bin/cu
	[sendemail]
	        smtpserver = /bin/m

	[diff "astextplain"]
	        textconv = astextpl
	[rebase]
	        autosquash = true

再次`rake deploy`可以发现，整个世界都清净了。