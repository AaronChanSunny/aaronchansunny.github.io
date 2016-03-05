---
layout: post
title: "Windows下修改输入法切换快捷键"
date: 2014-11-28 09:34:41 +0800
comments: true
tags: Android
keywords: input method, windows
description: Windows下修改输入法切换快捷键
---
## 引子
最近打算从Eclipse迁移到Android Studio（以下简称AS），这是Android开发的趋势。在AS中，代码自动补全的快捷键是`ctrl+space`，和Windows系统下切换输入法的快捷键冲突。既然这样，就更改输入法快捷键吧。当然方法很多，最方便的方法是直接修改注册表文件。<!-- more -->
## 过程
- 运行

![](/images/2014/11/26.png)

- 找到注册表项

![](/images/2014/11/27.png)

- 更改
修改Key Modifiers如下：

![](/images/2014/11/28.png)

- 重启或注销