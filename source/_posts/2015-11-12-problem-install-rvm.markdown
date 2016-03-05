---
layout: post
title: "记OSX环境安装RVM遇到的一个问题"
date: 2015-11-12 11:51:45 +0800
comments: true
tags: Octopress
---

之前都是在Windows平台搭建Octopress，这次在Mac上搭建，安装依赖项`Ruby 1.9.3`遇到一个问题。<!-- more -->

```
rvm install 1.9.3
```

遇到一个问题，异常信息如下：

```
[2015-11-12 10:20:49] ./configure
current path: /Users/ruijie/.rvm/src/ruby-1.9.3-p551
GEM_HOME=/Users/ruijie/.rvm/gems/ruby-2.2.1
PATH=/usr/local/opt/pkg-config/bin:/usr/local/opt/libtool/bin:/usr/local/opt/automake/bin:/usr/local/opt/autoconf/bin:/Users/ruijie/.rvm/gems/ruby-2.2.1/bin:/Users/ruijie/.rvm/gems/ruby-2.2.1@global/bin:/Users/ruijie/.rvm/rubies/ruby-2.2.1/bin:/Users/ruijie/Code/gitCode/pylearn2/pylearn2/scripts:/Users/ruijie/data/pylearn2/data:/opt/local/bin:/opt/local/sbin:/Users/ruijie/Documents/adt-bundle-mac-x86_64-20140702/sdk/platform-tools:/usr/local/Cellar/python/2.7.10_2/Frameworks/Python.framework/Versions/2.7/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/X11/bin:/Users/ruijie/.rvm/bin:/Users/ruijie/.rvm/bin
GEM_PATH=/Users/ruijie/.rvm/gems/ruby-2.2.1:/Users/ruijie/.rvm/gems/ruby-2.2.1@global
command(7): ./configure --prefix=/Users/ruijie/.rvm/rubies/ruby-1.9.3-p551 --with-opt-dir=/usr/local/opt/libyaml:/usr/local/opt/readline:/usr/local/opt/libksba:/usr/local/opt/openssl --without-tcl --without-tk --disable-install-doc --enable-shared
configure: WARNING: unrecognized options: --without-tcl, --without-tk
checking build system type... x86_64-apple-darwin14.5.0
checking host system type... x86_64-apple-darwin14.5.0
checking target system type... x86_64-apple-darwin14.5.0
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables... 
checking whether we are cross compiling... configure: error: in `/Users/ruijie/.rvm/src/ruby-1.9.3-p551':
configure: error: cannot run C compiled programs.
If you meant to cross compile, use `--host'.
See `config.log' for more details
```

通过关键信息`configure: error: cannot run C compiled programs.`可以看出，很可能是由于编译器导致的。谷歌搜索下，在[Can't install Ruby under Lion with RVM – GCC issues](http://stackoverflow.com/questions/8032824/cant-install-ruby-under-lion-with-rvm-gcc-issues)找到了答案。解决办法也很简单，手动指定编译器即可。

```
rvm install 1.9.3 --with-gcc=clang
```