---
layout: post
title: "ST使用Markdown Preview插件实现语法高亮和预览"
date: 2014-11-16 00:10:21 +0800
comments: true
tags: Tools
keywords: sublime text, markdown preview
description: Sublime Text使用Markdown Preview插件实现语法高亮和预览
---
## 引子
关于文本编辑器，一直热衷于Sublime Text（以下简称ST）。不仅仅因为它清爽的界面，更离不开它强大的插件支持。最近用Octpress搭建个人博客，开始学习Markdown标记语言。理所当然，马上想到用ST完成日常Markdown编写。随便打开一个Markdown文件，发现ST并不支持语法高亮。网上搜了下，找到了Markdown Preview，也就是本文的主角。<!-- more -->
##Markdown Preview安装
使用Package Control安装Markdown Preview非常简单。打开ST，按下组合键打开控制台:

	Control + `

> 注意：第二个组合键并不是单引号，而是Tab键的上一个按键。

出现控制台后，输入：

	import urllib2,os; pf='Package Control.sublime-package'; ipp = sublime.installed_packages_path(); os.makedirs( ipp ) if not os.path.exists(ipp) else None; urllib2.install_opener( urllib2.build_opener( urllib2.ProxyHandler( ))); open( os.path.join( ipp, pf), 'wb' ).write( urllib2.urlopen( 'http://sublime.wbond.net/' +pf.replace( ' ','%20' )).read()); print( 'Please restart Sublime Text to finish installation')

当看到代码最后一行提示的时候说明安装成功。  
此时重启ST，可在`Preferences -> Package Settings`看到`Package Control`。
## 快捷键
ST支持自定义快捷键，Markdown Preview默认没有快捷键。我们可以为Preview in Browser功能设置快捷键。打开`Preferences -> Key Bindings User`，在中括号内添加以下代码：

	{ "keys": ["alt+m"], "command": "markdown_preview", "args": { "target": "browser"} }

此处设置的快捷键为`"alt+m"`。
## 设置语法高亮和mathjax支持
打开`Preferences ->Package Settings->Markdown Preview->Setting Default`：  
查找`"enable_mathjax"`和`"enable_highlight"`，将其值更改为`true`。  
现在，开始尽情享受在ST下编写Markdown吧。