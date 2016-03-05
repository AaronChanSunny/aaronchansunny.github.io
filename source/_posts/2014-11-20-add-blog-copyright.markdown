---
layout: post
title: "给Octopress每篇博客添加版权信息"
date: 2014-11-20 11:03:51 +0800
comments: true
tags: Octopress
keywords: Octopress, copyright
description: 给Octopress每篇博客添加版权信息
---
## 引子
相信大多数人开博客的目的在于分享，在分享的过程中发现问题，共同进步。但是自己辛辛苦苦码字得到的一篇文章，还是希望能够打上自己的标签，这就涉及到如何给自己博客添加版权信息的问题。其实，Octopress高度模块化的设计理念，让我们很容易对自己的博客进行定制。我们要做的就是在响应模块，添加版权信息即可。<!-- more -->
## 添加自定义样式
Octopress所有样式都保存在`$OCTOPRESS_HOME/sass`目录下。在有自定义需求的地方，Octopress都为我们准备了`custom`子目录，例如：  
`$OCTOPRESS_HOME/sass/custom`  
`$OCTOPRESS_HOME/source/_includes/custom`  
现在，我们为版权信息模板添加一个自定义样式。  
打开`$OCTOPRESS_HOME/sass/custom/_styles.scss`文件，添加如下内容：

	.post-footer{margin-top:10px;padding:5px;background:none repeat scroll 0pt 0pt #eee;font-size:90%;color:gray}

这里，我们添加了一个自定义样式`post-footer`，我们将在版权信息模板中调用。
## 添加版权信息
我们希望为每篇博客都添加版权信息，因此需要在博客模板中修改。  
打开`$OCTOPRESS_HOME/source/_layouts/post.html`文件，在`<footer>`节点下添加内容：

	<p class="post-footer">
      {% if page.categories contains "reprints" %}
      {% else %}
          原文链接：
          <a href="http://aaronchansunny.github.io/{{page.url}}">http://aaronchansunny.github.io/{{page.url}}</a>
      {% endif %}
	</p>

解释下这段代码：当博客类别不属于`reprints`时（`reprints`属于转载的博客，转载自然不能添加版权信息啦！），Octopress将自动在博客末尾添加版权信息。该版权信息模块调用了之前我们自定义的`post-footer`样式，例子中仅仅附上原文链接，具体内容大家可以进行自定义。