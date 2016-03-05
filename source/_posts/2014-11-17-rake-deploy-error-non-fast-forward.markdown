---
layout: post
title: "解决Octopress提交博客异常"
date: 2014-11-17 14:55:51 +0800
comments: true
tags: Octopress
keywords: Git, rake deploy, non-fast-forward
description: 解决rake deploy错误
---
## 引子
今天在Octopress进行博客提交时，出现错误：  

	To git@github.com:aaronchansunny/aaronchansunny.github.com.git
	 ! [rejected]        master -> master (non-fast-forward)
	error: failed to push some refs to 'git@github.com:aaronchansunny/aaronchansunny.github.com.git'
	hint: Updates were rejected because the tip of your current branch is behind
	hint: its remote counterpart. Integrate the remote changes (e.g.
	hint: 'git pull ...') before pushing again.
	hint: See the 'Note about fast-forwards' in 'git push --help' for details.

从Git字面提示来看，应该是远程仓库和本地数据版本不一致引起的。那么，解决的方法就是让本地数据同步，因此pull远程数据merge后应该能解决问题。<!-- more -->
## 解决方法
首先，进入`_deploy`目录：

	OA@R06270 /e/octopress (source)
	$ cd _deploy/

接下来，pull远程仓库master分支数据：

	OA@R06270 /e/octopress/_deploy (master)
	$ git pull origin master

重新`rake deploy`：

	OA@R06270 /e/octopress/_deploy (master)
	$ cd ..
	OA@R06270 /e/octopress (source)
	$ rake deploy

问题解决。