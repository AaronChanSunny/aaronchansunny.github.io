---
layout: post
title: "在新机器上部署已有Octopress博客"
date: 2015-05-11 10:41:21 +0800
comments: true
tags: Octopress
---

通过[Github搭建Octopress博客](http://aaronchansunny.github.io/blog/2014/11/12/install-octopress/)，我们可以搭建Octopress。但是如果我们需要在一台新机器上部署Github上已有的Octopress，需要怎么做呢？步骤上大体一致。<!--more-->

## 克隆Github代码
分别克隆Github下**source**分支和**master**分支到本地。

- 克隆Source分支

``` 
$ git clone -b source git@github.com:username/username.github.com.git octopress
``` 

- 克隆Master分支

``` 
$ cd octopress 
$ git clone git@github.com:username/username.github.com.git _deploy
``` 

执行完这两步就OK了。

> 注意这里第2步一定要，不然在rake deploy时会报错：  
no such file or directory - _deploy

## 安装依赖项

另外，如果是重新在一台全新的电脑上要和服务器上的进行同步，除了上面的操作之外，还需要安装：

- Ruby
- Devkit
- Python
 
> 这里需要注意安装Ruby和Devkit的版本：  
> rubyinstaller-1.9.3-p550  
> DevKit-tdm-32-4.5.2-20111229-1559-sfx

安装完成后，运行如下命令：

``` ruby
$ cd DevKit 
$ ruby dk.rb init 
$ ruby dk.rb install
``` 

安装Python语法高亮模块：

``` python
$ cd Python34 
$ cd Scripts
$ easy_install pygments
``` 

进行Octopress依赖项安装：

``` 
$ cd octopress
$ gem install bundler
$ bundle install
``` 

> 如果出现SSL授权错误，请参考：  
[bundle install fails with SSL certificate verification error [duplicate]](http://stackoverflow.com/questions/10246023/bundle-install-fails-with-ssl-certificate-verification-error)

到此，Octopress环境就配置完成了，又能嗨皮地写博客了～
