---
layout: post
title: "Tomcat集群Session共享的轻量级解决方案Memcached"
date: 2014-11-19 16:27:56 +0800
comments: true
tags: Server
keywords: Tomcat, Session, memcached-session-manager
description: Tomcat集群Session共享的轻量级解决方案Memcached
---
## 引子
### Session是什么
用户访问服务器资源主要分成两类，一类是无状态访问，例如请求一张图片。另一类是有状态访问，这种情况下，服务器需要记录追踪用户状态，并根据用户所处状态做出不同响应，典型的例子是购物车。Session的作用就是在Web服务器上保持用户的状态信息。<!-- more -->
### Tomcat集群为什么需要Session共享
当客户端访问Tomcat集群时，所有的请求将被Nginx拦截，由Nginx做负载均衡后转发给后台真实Tomcat。按照这个流程就可能出现一个问题，当用户进行页面刷新或跳转时，每次请求将被转发给不同的Tomcat处理，这样就会造成Session的不同步。举个简单的栗子，例如当用户往购物车添加商品时，兴高采烈地准备买单了，当他跳转到付款页面却发现购物车被清空了，这就是Session丢失的典型栗子。因此，我们需要为集群环境做Session同步。
### Session共享方案讨论
在服务器集群的环境下，共享Session的方案主要分为4类：  

- 用户端本地保存Cookie  

在这种方式下，Web应用会将用户状态写到Cookie并保存到用户本地。但是，如果用户使用的浏览器不支持Cookie或者禁用Cookie，该方案将会失效。并且Cookie能保存的数据是有大小限制的，而且数据暴露给用户本地浏览器，存在安全性问题。 

- 采用数据库方式保存Session  

相对本地Cookie方式，将用户信息保存到服务端数据库解决了数据安全性问题。然而，这么做是有代价的，应用中所有对Session的访问都必须经过数据库，加大数据库负担，导致系统整体性能降低。  

- 代理服务器  

通过代理服务器实现Session共享的思路非常简单，就是Session数据在哪台Tomcat，之后的请求都转发到这台Tomcat。例如Nginx，具体实现只需要修改转发规则为`ip_hash`即可。但这时候可能存在某一时间段大量用户始终访问某台Tomcat的负载很大，也就失去了负载均衡的意义。  

- 搭建缓存服务器  

这种方案也是应用最普遍的方案，通过搭建缓存服务器，并使用第三方工具接管Tomcat对Session的管理。  
本文例子采用方案4进行Session管理，使用的缓存服务器是Memcached，并使用memcached-session-manager管理Session。memcached-session-manager（以下简称msm）的使用方法很简单，只需要根据Tomcat版本和序列化方式下载相应jar包，拷贝至Tomcat的lib目录下，最后修改Tomcat配置文件，更换Session管理模块即可。
## 原型环境
- 1XX.XX.XX.181  

运行Nginx、Tomcat和Tomcat1  

- 1XX.XX.XX.182   

运行Memcached  
其中1XX.XX.XX.181主机搭建Tomcat集群环境，1XX.XX.XX.182主机搭建缓存服务器。  
服务器环境： 

	# cat /etc/redhat-release 
	CentOS release 6.5 (Final)
	# uname -a
	Linux chenllcentos.localdomain 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

## 安装Memcached
[Memcached](http://memcached.org/)是一个高性能的分布式缓存系统，进入[下载页面](http://memcached.org/downloads)下载最新稳定版本。或者，可以通过`wget`下载：

	# wget http://memcached.org/latest

安装Memcached之前，需要安装libevent-devel依赖：

	# yum -y install libevent-devel

解压并安装：

	# tar -zxvf memcached-1.x.x.tar.gz
	# cd memcached-1.x.x
	# ./configure && make && make test && sudo make install

启动`memcached`运行命令：

	# /usr/local/memcached-1.4.21/bin/memcached -d -uroot -m 1024 -P /usr/local/memcached-1.4.21/memcached.pid

这里`-d`参数表示开启守护线程，`-u`参数指定用户，`-m`参数指定分配给`memcached`的内存大小。更多启动参数如下：

	-d 选项是启动一个守护进程
	-m 是分配给Memcache使用的内存数量，单位是MB，这里是1024MB，默认是64MB
	-u 是运行Memcache的用户，这里是root
	-l 是监听的服务器IP地址，默认应该是本机
	-p 是设置Memcache监听的端口，默认是11211，最好是1024以上的端口
	-c 选项是最大运行的并发连接数，默认是1024，这里设置了10240，按照你服务器的负载量来设定
	-P 是设置保存Memcache的pid文件位置
	-h 打印帮助信息
	-v 输出警告和错误信息
	-vv 打印客户端的请求和返回信息

查看端口状态：  

	[root@chenllcentos bin]# netstat -ntlp | grep memcached
	tcp        0      0 0.0.0.0:11211               0.0.0.0:*                   LISTEN      2222/memcached      
	tcp        0      0 :::11211                    :::*                        LISTEN      2222/memcached      

在集群环境中，Tomcat需要访问缓存服务器读取并更新Session信息，因此缓存服务器需要对`11211`端口放行：

	# vi /etc/sysconfig/iptables

添加如下内容：

	# 放行Memcached端口
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 11211 -j ACCEPT

重启`iptables`：

	# service iptables restart

停止`memcached`通过`kill`命令：

	# kill `cat /usr/local/memcached-1.4.21/memcached.pid`

## Tomcat配置msm
msm是托管在GoogleCode的一个开源项目，项目详细介绍可以查阅[Tomcat high-availability clusters with memcached](https://code.google.com/p/memcached-session-manager/)。
### 拷贝jar包
使用msm之前，需要拷贝一下jars到`$TOMCAT_HOME/lib`目录：  
[memcached-session-manager-${version}.jar](http://repo1.maven.org/maven2/de/javakaffee/msm/memcached-session-manager/)  
[memcached-session-manager-tc8-${version}.jar](http://repo1.maven.org/maven2/de/javakaffee/msm/memcached-session-manager-tc8/)  

> 该jar包需要和Tomcat版本一致。  

[spymemcached-2.11.1.jar](http://repo1.maven.org/maven2/net/spy/spymemcached/2.11.1/spymemcached-2.11.1.jar)  
这里选择`kryo-serializer`作为序列化策略，因此还需要添加如下用于实现对象序列化的jar包:   
[msm-kryo-serializer](http://repo1.maven.org/maven2/de/javakaffee/msm/msm-kryo-serializer/)  
[kryo-serializers-0.11](http://repo1.maven.org/maven2/de/javakaffee/kryo-serializers/0.11/)  
[kryo](http://repo1.maven.org/maven2/com/googlecode/kryo/)  
[minlog](http://repo1.maven.org/maven2/com/googlecode/minlog/)  
[reflectasm](http://repo1.maven.org/maven2/com/googlecode/reflectasm/)  
[asm-3.2](http://repo1.maven.org/maven2/asm/asm/3.2/)  
最后，还需要添加`javolution-serializer`所需jar包：  
[msm-javolution-serializer](http://repo1.maven.org/maven2/de/javakaffee/msm/msm-javolution-serializer/)  
[javolution-5.4.3.1](http://memcached-session-manager.googlecode.com/svn/maven/javolution/javolution/5.4.3.1/)  
### 修改Tomcat配置文件
修改`$TOMCAT_HOME/conf/context.xml`配置文件，新增如下信息：

	<Context>
		...
	    <!-- Memcached管理集群session  -->
	    <Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager"
	          memcachedNodes="n1:1XX.XX.XX.182:11211"
	          sticky="false"
	          lockingMode="auto"
	          sessionBackupAsync="false"
	          sessionBackupTimeout="1000"
	          transcoderFactoryClass="de.javakaffee.web.msm.serializer.kryo.KryoTranscoderFactory"  />
	    ...
	</Context>

> 几点说明：  
- 其中memcachedNodes表示memcached节点信息,多个节点以空隔分开；  
- sticky指定Session共享模式；  
- sessionBackupTimeout的单位为分钟。

分别启动主机1XX.XX.XX.181的Nginx、Tomcat和Tomcat1，启动主机1XX.XX.XX.182的Memcached服务器。
## Session共享测试
为了测试Nginx集群是否实现session共享，让index.jsp打印Session信息：

	SessionID:<%=session.getId()%>

打开浏览器访问，如果配置正确可得到如下以`-n1`为结尾的Session串：

	SessionID:2F712A985872A8688138D293F59E493A-n1.tomcat 

刷新浏览器，可以发现Session并不会发生变化，说明通过Memcached实现Session共享有效。如果我们关闭主机1XX.XX.XX.182的缓存服务器呢？

	# kill `cat /usr/local/memcached-1.4.21/memcached.pid`

再刷新浏览器看看，将出现类似如下格式的Session串：

	SessionID:E914205336C4FE5A386702F2B6D65063.tomcat 

> 注意：
如果刷新后仍然出现以以`-n1`为结尾的Session串，等待几秒后再试试。这是因为关闭缓存服务器需要时间，如果马上刷新Tomcat集群还是能从缓存服务器取到Session。

可以发现`-n1`结尾消失了，说明Session共享已经失效，并且反复刷新浏览器，Session的值在不断变化。