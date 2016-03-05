---
layout: post
title: "Git删除远程仓库文件"
date: 2014-11-17 15:58:33 +0800
comments: true
tags: Git
keywords: Git, delete, remote
description: Git删除远程仓库文件
---
## 引子
假设这么一个场景：如果已经提交某文件到远程仓库，但现在本地已经删除该文件了。<!-- more -->  
这种情况下，输入`git status`将提示以下信息：

	$ git status
	On branch source
	Your branch is up-to-date with 'origin/source'.

	Changes to be committed:
	  (use "git reset HEAD <file>..." to unstage)

	        deleted:    source/_posts/2014-11-11-master-slave-transfer.html

其中`source/_posts/2014-11-11-master-slave-transfer.html`文件就是本地已经删除，但远程仓库还存在。怎么同步远程仓库数据库呢？  
此时使用`git add .`并不会在索引中删除这个文件，而要使用带`-a`选项的git commit命令和带`-A`选项的git add命令来完成。  

	$ git add -A
	$ git commit -a -m "delete file 2014-11-11-master-slave-transfer.html"
	$ git push origin source

提交完成后，再次`git status`：

	$ git status
	On branch source
	Your branch is up-to-date with 'origin/source'.

	nothing to commit, working directory clean

可以发现，Git状态恢复正常，现在就可以继续使用`git add .`进行添加了。