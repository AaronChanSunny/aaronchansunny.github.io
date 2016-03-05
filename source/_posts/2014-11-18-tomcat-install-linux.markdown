---
layout: post
title: "Linux安装Tomcat并配置多实例"
date: 2014-11-18 13:43:26 +0800
comments: true
tags: Server
keywords: Linux, Tomcat, Install
description: Linux安装Tomcat并配置多实例
---
## 引子
当考虑部署JavaWeb项目时，选择Linux服务器是大多数人的选择。Web应用必须在Web容器中运行，目前对于Web容器的选择有很多：Apache、Tomcat、IIS、Nginx和GWS等。其中Tomcat可以说是为JavaWeb应用量身定做的Web容器，又因为其轻巧并且功能强大，往往是中小型项目最好的选择。如果运用代理服务器做Tomcat集群，甚至能轻松搭建出具有高并发性能的大型Web应用。<!-- more -->
## 安装准备
安装环境：  

	[root@chenllcentos ~]# cat /etc/redhat-release 
	CentOS release 6.5 (Final)
	[root@chenllcentos ~]# uname -a
	Linux chenllcentos.localdomain 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

安装包：  
[jdk-8u25-linux-x64](http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jdk-8u25-linux-x64.tar.gz)  
[apache-tomcat-8.0.15](http://mirror.bit.edu.cn/apache/tomcat/tomcat-8/v8.0.15/bin/apache-tomcat-8.0.15.tar.gz)
##具体流程
Tomcat运行需要JDK环境支持，因此首先需要安装配置好JDK。
### 安装JDK
安装JDK的步骤非常简单，和Windows环境下的过程类似，分成安装和配置环境变量两个步骤。这里采用源码安装的方式进行说明。  
创建安装目录：

	[root@chenllcentos ~]# mkdir -p /usr/local/java

进入JDK安装包所在目录，解压安装包：
	
	[root@chenllcentos ~]# cd ~/dev
	[root@chenllcentos dev]# tar zxvf jdk-8u25-linux-x64.tar.gz

复制文件至安装目录：

	[root@chenllcentos dev]# cp -rf jdk1.8.0_25/ /usr/local/java 

接下来需要配置Java环境变量，打开`etc/profile`文件：

	[root@chenllcentos dev]# vi /etc/profile

在文件末尾添加如下内容：

	export JAVA_HOME=/usr/local/java
	export CLASS_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib
	export PATH=.:$PATH:$JAVA_HOME/bin

重新加载`etc/profile`文件使其生效：

	[root@chenllcentos dev]# source /etc/profile

验证JDK安装是否成功：

	[root@chenllcentos ~]# java -version
	java version "1.8.0_25"
	Java(TM) SE Runtime Environment (build 1.8.0_25-b17)
	Java HotSpot(TM) 64-Bit Server VM (build 25.25-b02, mixed mode)

至此，JDK安装完毕。  
成功安装JDK后，可以进行Tomcat安装，这里同样采用源码安装的方式进行说明。
### 安装Tomcat
Tomcat源码安装和JDK安装过程类似，同样是分成安装和配置环境变量两个步骤。  
创建Tomcat安装目录：

	[root@chenllcentos ~]# mkdir -p /usr/local/tomcat

进入Tomcat安装包目录，解压安装包：

	[root@chenllcentos ~]# cd ~/dev
	[root@chenllcentos dev]# tar zxvf apache-tomcat-8.0.15.tar.gz

复制文件至Tomcat安装目录：

	[root@chenllcentos dev]# cp -rf apache-tomcat-8.0.15/ /usr/local/tomcat 

接下来同样需要为Tomcat配置环境变量，打开`etc/profile`文件：

	[root@chenllcentos ~]# vi /etc/profile

在文件末尾添加如下内容：

	export CATALINA_HOME=/usr/local/tomcat
	export CATALINA_BASE=/usr/local/tomcat
	export CATALINA_TMPDIR=/usr/local/tomcat/tmp

重新加载`etc/profile`文件使其生效：

	[root@chenllcentos ~]# source /etc/profile

至此，Tomcat安装完成。现在我们试试启动Tomcat：

	[root@chenllcentos ~]# /usr/local/tomcat/bin/startup.sh 
	Using CATALINA_BASE:   /usr/local/tomcat
	Using CATALINA_HOME:   /usr/local/tomcat
	Using CATALINA_TMPDIR: /usr/local/tomcat/tmp
	Using JRE_HOME:        /usr/local/jdk1.8.0_25
	Using CLASSPATH:       /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar
	Tomcat started.

根据提示，Tomcat已经成功启动。我们通过`ps`命令查看进程验证下：

	[root@chenllcentos ~]# ps -ef | grep tomcat
	root     30971     1 54 14:26 pts/0    00:00:31 /usr/local/jdk1.8.0_25/bin/java -Djava.util.logging.config.file=/usr/local/tomcat/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.endorsed.dirs=/usr/local/tomcat/endorsed -classpath /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar -Dcatalina.base=/usr/local/tomcat -Dcatalina.home=/usr/local/tomcat -Djava.io.tmpdir=/usr/local/tomcat/tmp org.apache.catalina.startup.Bootstrap start
	root     31001  8578  0 14:27 pts/0    00:00:00 grep tomcat

可以看出，Tomcat确实已经启动并在后台运行。现在，我们通过`netstat`查看端口监听状态：

	[root@chenllcentos ~]# netstat -ntlp | grep java
	tcp        0      0 :::8080                     :::*                        LISTEN      30971/java          
	tcp        0      0 ::ffff:127.0.0.1:8005       :::*                        LISTEN      30971/java          
	tcp        0      0 :::8009                     :::*                        LISTEN      30971/java    

根据结果显示，Tomcat进程（PID:30971）一共监听了3个端口。其中`8080`端口大家都知道，是Tomcat默认监听端口。但是其他2个端口呢？不着急，让我们看看Tomcat配置文件`server.xml`是怎么解释的。

	[root@chenllcentos ~]# cat /usr/local/tomcat/conf/server.xml | grep port
	<Server port="8005" shutdown="SHUTDOWN">
	         Define a non-SSL HTTP/1.1 Connector on port 8080
	    <Connector port="8080" protocol="HTTP/1.1"
	               port="8080" protocol="HTTP/1.1"
	    <!-- Define a SSL HTTP/1.1 Connector on port 8443
	    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
	    <!-- Define an AJP 1.3 Connector on port 8009 -->
	    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
	    <!-- You should set jvmRoute to support load-balancing via AJP ie :

这下就清楚了，原来`8005`端口是Tomcat关闭端口，而`8009`是`AJP/1.3`协议端口。其实，Tomcat单机运行多个Tomcat实例的原理就是通过修改Tomcat监听端口实现的。
关闭Tomcat：

	[root@chenllcentos dev]# /usr/local/tomcat/bin/shutdown.sh 
	Using CATALINA_BASE:   /usr/local/tomcat
	Using CATALINA_HOME:   /usr/local/tomcat
	Using CATALINA_TMPDIR: /usr/local/tomcat/tmp
	Using JRE_HOME:        /usr/local/jdk1.8.0_25
	Using CLASSPATH:       /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar

## 多实例配置
如果Tomcat所部署的主机硬件性能较高，单个Tomcat实例可能造成机器性能的浪费。因此，可以考虑通过单机运行多个Tomcat实例来达到充分利用服务器物理资源的目的。为Tomcat分配不同的监听端口和启动目录即可实现单机多实例。
### 复制安装文件
首先，我们为新实例创建一个安装目录`tomcat1`：

	[root@chenllcentos ~]# mkdir -p /usr/local/tomcat1

首先，需要为Tomcat实例复制Tomcat安装文件：

	[root@chenllcentos dev]# cp -rf /usr/local/tomcat /usr/local/tomcat1

### 修改监听端口
修改tomcat1的server.xml文件，分别修改`SHUTDOWN`、`HTTP/1.1`和`AJP/1.3`三个端口，修改后如下：

	[root@chenllcentos dev]# cat /usr/local/tomcat1/conf/server.xml | grep port
	<Server port="8006" shutdown="SHUTDOWN">
	         Define a non-SSL HTTP/1.1 Connector on port 8080
	    <Connector port="8081" protocol="HTTP/1.1"
	               port="8081" protocol="HTTP/1.1"
	    <!-- Define a SSL HTTP/1.1 Connector on port 8443
	    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
	    <!-- Define an AJP 1.3 Connector on port 8009 -->
	    <Connector port="8010" protocol="AJP/1.3" redirectPort="8443" />
	    <!-- You should set jvmRoute to support load-balancing via AJP ie :

最后一步，还需要修改启动和关闭脚本的Tomcat环境变量。  
### 更新环境变量
修改Tomcat新实例的`catalina.sh`脚本：

	 [root@chenllcentos ~]# vi /usr/local/tomcat1/bin/catalina.sh

在文件的开头添加如下内容：

	export CATALINA_HOME=/usr/local/tomcat1
	export CATALINA_BASE=/usr/local/tomcat1
	export CATALINA_TMPDIR=/usr/local/tomcat1/tmp

> 注意：此处必须在开头部分添加Tomcat环境变量，否则环境变量无法生效，导致多实例启动失败。

有些朋友可能有疑问了，我们是通过`startup.sh`和`shutdown.sh`来启动和关闭Tomcat的，这里为什么在`catalina.sh`修改环境变量就能生效呢？  
其实原因很简单，在执行`startup.sh`和`shutdown.sh`这两个脚本时，会调用`catalina.sh`脚本。让我们验证下：

	[root@chenllcentos ~]# cat /usr/local/tomcat1/bin/startup.sh | grep catalina.sh
	EXECUTABLE=catalina.sh
	[root@chenllcentos ~]# cat /usr/local/tomcat1/bin/shutdown.sh | grep catalina.sh
	EXECUTABLE=catalina.sh

所以，我们只需要在`catalina.sh`脚本中配置Tomcat环境即可。  
### 运行多实例
现在启动我们刚才配置的Tomcat新实例：  

	[root@chenllcentos ~]# /usr/local/tomcat1/bin/startup.sh

查看进程：

	[root@chenllcentos ~]# ps -ef | grep java
	root     31163     1 99 15:21 pts/0    00:00:25 /usr/local/jdk1.8.0_25/bin/java -Djava.util.logging.config.file=/usr/local/tomcat1/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.endorsed.dirs=/usr/local/tomcat1/endorsed -classpath /usr/local/tomcat1/bin/bootstrap.jar:/usr/local/tomcat1/bin/tomcat-juli.jar -Dcatalina.base=/usr/local/tomcat1 -Dcatalina.home=/usr/local/tomcat1 -Djava.io.tmpdir=/usr/local/tomcat1/tmp org.apache.catalina.startup.Bootstrap start
	root     31200     1 99 15:21 pts/0    00:00:13 /usr/local/jdk1.8.0_25/bin/java -Djava.util.logging.config.file=/usr/local/tomcat/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.endorsed.dirs=/usr/local/tomcat/endorsed -classpath /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar -Dcatalina.base=/usr/local/tomcat -Dcatalina.home=/usr/local/tomcat -Djava.io.tmpdir=/usr/local/tomcat/tmp org.apache.catalina.startup.Bootstrap start
	root     31224  8578  0 15:21 pts/0    00:00:00 grep java

查看端口状态：

	[root@chenllcentos ~]# netstat -ntlp | grep java
	tcp        0      0 :::8080                     :::*                        LISTEN      31200/java          
	tcp        0      0 :::8081                     :::*                        LISTEN      31163/java          
	tcp        0      0 ::ffff:127.0.0.1:8005       :::*                        LISTEN      31200/java          
	tcp        0      0 ::ffff:127.0.0.1:8006       :::*                        LISTEN      31163/java          
	tcp        0      0 :::8009                     :::*                        LISTEN      31200/java          
	tcp        0      0 :::8010                     :::*                        LISTEN      31163/java 

单机运行多个Tomcat实例配置成功。
## 后续
到这里，Tomcat的安装和部署就全部完成了。此时可以通过浏览器访问Tomcat[默认主页](http://localhost:8080/)，不出意外都能见到可爱的小猫。那么，如果是通过远程主机的方式访问呢？  
服务器IP地址；

	[root@chenllcentos ~]# ifconfig
	eth0      Link encap:Ethernet  HWaddr 52:54:00:11:45:2A  
	          inet addr:1XX.XX.XX.181  Bcast:1XX.XX.XX.255  Mask:255.255.255.128
	......

结果发现，通过[http://1XX.XX.XX.181:8080/](http://1XX.XX.XX.181:8080/)并不能访问。原来出于服务器安全考虑，Linux自己一套防火墙机制，默认不对应用端口放行，防止非法的服务器攻击。因此，如果要使Tomcat能被远程访问，还要设置Linux防火墙，对我们配置的`HTTP/1.1`端口放行。  
打开`iptables`文件：

	[root@chenllcentos ~]# vi /etc/sysconfig/iptables

添加如下内容：

	# 放行Tomcat8080端口
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT
	# 放行Tomcat18081端口
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 8081 -j ACCEPT

重启防火墙：

	[root@chenllcentos ~]# service iptables restart
	iptables：将链设置为政策 ACCEPT：filter                    [确定]
	iptables：清除防火墙规则：                                 [确定]
	iptables：正在卸载模块：                                   [确定]
	iptables：应用防火墙规则：                                 [确定]

现在，Tomcat就能支持远程主机访问了。