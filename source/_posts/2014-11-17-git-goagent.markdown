---
layout: post
title: "给Git设置goagent代理"
date: 2014-11-17 11:06:44 +0800
comments: true
tags: Git
keywords: Git, goagent, Proxy
description: 给Git设置goagent代理
---
## 引子
最近使用Git提交代码到仓库，经常报权限错误。<!-- more -->

	$ git push origin source
	Read from socket failed: Connection reset by peer
	fatal: Could not read from remote repository.

	Please make sure you have the correct access rights
	and the repository exists.

排查之后，权限根本没有问题，剩下的最大可能就是我朝防火墙问题了。浏览器方面一直使用的是goagent，网上关于goagent教程都是针对浏览器设置的。其实只要对代理的运行原理稍微了解的同学就知道，代理都是通过配置监听端口的方式实现的，goagent当然也不例外。因此，理论上任何应用只要配置了代理主机端口，就能正常使用代理。启动goagent可以看到如下信息：

	------------------------------------------------------
	GoAgent Version    : 3.1.19 (python/2.7.6 gevent/1.0.1 pyopenssl/0.13)
	Uvent Version      : 0.3.0 (pyuv/0.11.0-dev libuv/0.11.2-pre)
	Listen Address     : 127.0.0.1:8087
	GAE Mode           : https
	GAE Profile        : ipv4
	GAE APPID          : aaronchansunny
	Pac Server         : http://172.24.128.249:8086/proxy.pac
	Pac File           : file://E:\下载\goagent-goagent-3591602\local\proxy.pac
	------------------------------------------------------

可以知道，goagent监听端口为：`127.0.0.1:8087`。那么，剩下的事情就简单了，只要为Git设置代理信息，让Git协议走goagent不就能解决防火墙问题了么。
## 设置Git代理
设置Git代理非常简单，让`https_proxy`和`http_proxy`走goagent即可。  
打开Git Bash运行命令：

	$ git config --global http.proxy 127.0.0.1:8087
	$ git config --global https.proxy 127.0.0.1:8087
	$ git config --global http.sslVerify false

查看Git设置信息：

	$ git config --list
	......
	http.sslverify=true
	http.proxy=127.0.0.1:8087
	https.proxy=127.0.0.1:8087
	......

此时通过`http`或`https`协议`git clone`走的就是goagent代理。  
若想取消代理呢？

	$ git config --global --unset http.proxy
	$ git config --global --unset https.proxy

此时再查看Git设置信息，就看不到之前的代理设置了。
其实，通过`git config`添加设置信息，都是修改`~/.gitconfig`文件，上述代理设置也可以直接通过修改该配置文件来完成。

	$ vi ~/.gitconfig

配置文件内容如下：

	[user]
	        name = aaronchansunny
	        email = aaronchansunny@gmail.com
	[core]
	        autocrrlf = false
	[http]
	        sslVerify = true
	        proxy = 127.0.0.1:8087
	[https]
	        proxy = 127.0.0.1:8087

可以看到，之前的操作相当于在`[http]`和`[https]`新增相应代理信息。