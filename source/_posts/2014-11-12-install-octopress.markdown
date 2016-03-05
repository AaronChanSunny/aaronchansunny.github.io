---
layout: post
title: "Github搭建Octopress博客"
date: 2014-11-12 19:22:01 +0800
comments: true
tags: Octopress
keywords: github, octopress
description: Github搭建Octopress博客
---
## 引子
一直想要有个个人博客。在学校的时候觉得别人用Wordpress搭了个个人网站，那感觉牛X哄哄啊。工作以来，成了Github粉，偶然机会知道可以用Octopress在Github上面搭建个人博客，还能用命令行写博客，听起来就蠢蠢欲动。心动不如行动，让我们都像写程序一样写博客吧。<!-- more -->
## 一些唠叨
上网找了一些资料，大家都说搭建Github-Octopress博客门槛很高，不仅要熟悉Git命令，还要会Ruby，会Python，你要写博客还要学Markdown语法。其实都不是，只要你喜欢，尽管尝试就好了。期间肯定会遇到各种问题，首先是看错误日志，很多时候都是因为缺少依赖包导致的。要是自己解决不了，请教最好的老师，谷歌爸爸。以下例子，基于Windows操作系统。
## 搭建过程
### 准备工作
你需要有一个Github账号，怎么注册自行搜索。有一点要注意，注册完Github一定要验证邮箱。笔者一开始就是因为没有验证邮箱，导致`rake deploy`后始终404，折腾了好一阵子。
### 安装Ruby
下载[rubyinstaller-1.9.3-p550](http://pan.baidu.com/s/1hqBtzi0)，安装时记得选中`Add Ruby executables to your PATH`。
### 安装Devkit
下载[DevKit-tdm-32-4.5.2-20111229-1559-sfx](http://pan.baidu.com/s/1c0GRozu)，下载完成后，将其解压到磁盘根目录，例如E:DevKit，然后CMD执行如下命令：

	C:\Users\OA> E: 
	E:\> cd DevKit 
	E:\DevKit> ruby dk.rb init 
	E:\DevKit> ruby dk.rb install

### 安装Python
下载[python-3.4.2](http://pan.baidu.com/s/1dDENsTr)，一路next即可。博客的代码高亮用到了Python的Pygments模块，在Python中安装第三方库需要使用easy_install，easy_install会安装在Python安装目录的Scripts目录中，例如我的Python目录是E:\Python34，执行命令：

	C:\Users\OA> E: 
	E:\> cd Python34 
	E:\Python34>cd Scripts
	E:\Python34\Scripts> easy_install pygments

### 安装Octopress
首先将Octopress代码拉到本地：

	C:\Users\OA> E: 
	E:\> git clone git://github.com/imathis/octopress.git octopress

然后需要安装Octopress的依赖项，安装依赖项需要用到Ruby的gem，使用国内的淘宝镜像速度相对快点：

	E:\> cd octopress
	E:\octopress> gem sources -a http://ruby.taobao.org/
	E:\octopress> gem sources -r http://rubygems.org/
	E:\octopress> gem sources -l

进行Octopress依赖项安装：

	E:\octopress> gem install bundler
	E:\octopress> bundle install

### 中文编码问题
为了支持中文UTF-8编码，对Windows环境变量配置如下：

	LANG=zh_CN.UTF-8 
	LC_ALL=zh_CN.UTF-8

### 新建Github仓库
登录Github，假设你的用户名是username，首先要新建一个命名如下的Repo：

> username.github.com

命名必须是这个格式，否则在运行命令`rake setup_github_pages`之后将不能够自动创建后面提到的master和source分支。
### 发布Octopress到Github
在Octopress目录，运行命令：

	E:\octopress> rake setup_github_pages

按照提示输入刚才新建的GitHub Repo SSH地址，格式类似：  

> git@github.com:username/username.github.com.git  

安装Octopress的默认主题：

	E:\octopress> rake install

生成静态文件：

	E:\octopress> rake generate

到此所有的安装工作已经结束，输入下面的命令可以在本地进行预览：

	E:\octopress> rake preview

当完成一篇博客或更新博客配置之后，想预览下效果，可以使用`rake preview`进行本地预览。运行命令后，看屏幕上提示，按Ctrl+C并输入Y来终止批处理操作。然后打开浏览器，通过[预览地址](http://localhost:4000/)进行浏览。如果效果不满意，修改源文件重新运行`rake generate`即可。  
以上工作都完成后，就可以把博客发布到Github上。诸如数据库资源，服务器资源等搭建个人博客所需要的资源，你能想到的一切Github都为你提供，当然还有版本控制。将博客发布到Github上，运行命令：

	E:\octopress> rake deploy

## Ocotpress基本配置
默认的博客运行成功的话，就需要按照自己的要求对博客配置进行修改了，主要是修改Octopress根目录下的主配置文件`_config.yml`。

	url:  http://username.github.com                 # 博客地址

	title:                                                       # 博客标题 
	subtitle:                                                  # 副标题 
	author:                                                   # 作者 
	simple_search:  http://www.google.com.hk/search     # 搜索引擎 
	description:                                                            # 关于博客的描述 
	subscribe_rss:  /atom.xml                  # Rss订阅地址, 默认是  /atom.xml 
	subscribe_email:                               # 提供Email订阅的地址 
	email:                                              # Rss订阅的Email地址

	root:  /               # 博客路径，默认是“/“，如果你打算在子目录中，记得修改这个路径 
	permalink: /blog/:year/:month/:day/:title/           # 文章的固定链接形式

## 后续
当然，你还能为你的博客配置更多，例如导航栏汉化，侧边栏定制，评论系统等。具体怎么做，都可以请教谷歌爸爸。  
写这篇博客的时候，已经使用Octopress将近两周了。由于对Git命令不熟悉，刚开始几天各种不适应，但现在已经能够愉快地写博客了。其实最初选择Octopress的动机非常单纯，因为Github嘛。搭个博客记录平时学习的内容，还能顺便熟悉Git命令，何乐而不为。