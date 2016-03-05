---
layout: post
title: "编译安装Percona Server并使用Transfer实现主从数据库同步加速（上）"
date: 2014-11-06 14:10:58 +0800
comments: true
tags: Database
keywords: percona server, mysql, transfer
description: 编译安装Percona Server并使用Transfer实现主从数据库同步加速（上）
---
## 折腾缘由
今天机缘巧合和同事聊起Mysql主从复制，经他点拨才知道从数据库同步数据居然只是是单线程，当主数据库写入数据量庞大，数据同步延迟将十分明显。显然，这在生产环境中是不被允许的。既然知道主从同步延迟的根源在与单线程，那么思路就来了，采取多线程不就行了么？上网搜了下，[追风刀·丁奇](http://dinglin.iteye.com/)一直关注这个问题，并提供Mysql-Transfer作为Mysql原生主从同步延时的解决方案。说干就干，站在巨人的肩膀上。<!-- more -->  
由于Transfer本质是一个在Mysql源码上打的Patch，因此既可以当作Slave，也可以当成第三方工具，负责将Master的数据同步发给Slave。最新版本Transfer是基于Percona 5.5.34，而通过yum安装的Mysql版本为5.1。

	[root@chenllcentos ~]# rpm -aq | grep mysql
	mysql-5.1.73-3.el6_5.x86_64
	mysql-devel-5.1.73-3.el6_5.x86_64
	mysql-libs-5.1.73-3.el6_5.x86_64
	mysql-server-5.1.73-3.el6_5.x86_64

因此考虑编译安装对Mysql进行升级。至于使用yum安装Mysql和初始配置的基本步骤，可参考上一篇博客[Mysql主从复制和读写分离方案分析](http://localhost:4000/blog/2014/11/05/mysql-read-write/)。
## 编译安装Percona Server
### 安装前准备

- 卸载Rpm安装的Mysql  

检查是否安装有Mysql。
	
	[root@chenllcentos ~]# rpm -aq | grep mysql

若已安装则通过如下命令卸载，否则跳过此步骤。

	[root@chenllcentos ~]# rpm -e mysql mysql-devel mysql-libs mysql-server #普通删除模式  
	[root@chenllcentos ~]# rpm -e --nodeps mysql mysql-devel mysql-libs mysql-server  #强行删除模式

- 安装编译环境

	[root@chenllcentos ~]# yum -y install make gcc-c++ cmake bison-devel ncurses ncurses-devel

请确认以上组件都已经安装，否则编译安装Percona Server将不成功。若出现问题怎么办，找谷歌爸爸，相信都能找到解决办法。  

- 下载Percona Server

[Percona-Server-5.5.34-rel32.0.tar.gz](http://www.percona.com/redir/downloads/Percona-Server-5.5/Percona-Server-5.5.34-rel32.0/source/Percona-Server-5.5.34-rel32.0.tar.gz)，若下载失败，请使用[备用地址](http://pan.baidu.com/s/1pJsikd1)。  
或者，在Linux下直接使用wget命令：

	[root@chenllcentos dev]# wget  http://www.percona.com/redir/downloads/Percona-Server-5.5/Percona-Server-5.5.34-rel32.0/source/Percona-Server-5.5.34-rel32.0.tar.gz

下载失败的话，多尝试几次。接下来，步入正题，进行Percona Server编译安装。

### 具体步骤

Percona Server可以简单理解成Mysql编译安装和Mysql编译安装过程相同。

- 创建mysql的用户与组

	[root@mysql ~]# groupadd -r mysql
	[root@mysql ~]# useradd -g mysql -r -s /sbin/nologin mysql

若系统提示：

	[root@chenllcentos ~]# groupadd -r mysql
	groupadd: group 'mysql' already exists
	[root@chenllcentos ~]# useradd -g mysql -r -s /sbin/nologin mysql
	useradd: user 'mysql' already exists

说明该服务器之前已安装过Mysql数据库，用户组和用户已存在，直接进入下一步骤即可。  
解压Percona Server安装包。

	[root@chenllcentos dev]# tar zxvf Percona-Server-5.5.34-rel32.0.tar.gz

进入Percona Server目录，并配置编译环境。
	
	[root@chenllcentos dev]# cd Percona-Server-5.5.34-rel32.0
	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# CFLAGS="-O3 -g -fno-exceptions -static-libgcc -fno-omit-frame-pointer -fno-strict-aliasing"
	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# CXX=gcc
	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# CXXFLAGS="-O3 -g -fno-exceptions -fno-rtti -static-libgcc -fno-omit-frame-pointer -fno-strict-aliasing"
	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# export CFLAGS CXX CXXFLAGS

接下来，使用cmake进行编译。

	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DEXTRA_CHARSETS=all    \
	-DCMAKE_BUILD_TYPE:STRING=Release             \
	-DSYSCONFDIR:PATH=/etc           \
	-DENABLED_PROFILING:BOOL=ON                   \
	-DENABLE_DEBUG_SYNC:BOOL=OFF                  \
	-DMYSQL_DATADIR:PATH=/usr/local/mysql/data    \
	-DMYSQL_MAINTAINER_MODE:BOOL=OFF              \
	-DWITH_EXTRA_CHARSETS:STRING=all  \
	-DWITH_BIG_TABLES:BOOL=ON \
	-DWITH_FAST_MUTEXES:BOOL=ON \
	-DENABLE-PROFILING:BOOL=ON \
	-DWITH_SSL:STRING=bundled                     \
	-DWITH_UNIT_TESTS:BOOL=OFF                    \
	-DWITH_ZLIB:STRING=bundled                    \
	-DWITH_PARTITION_STORAGE_ENGINE:BOOL=ON       \
	-DWITH_PLUGINS=heap,csv,partition,innodb_plugin,myisam \
	-DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_EXTRA_CHARSETS=ALL \
	-DENABLED_ASSEMBLER:BOOL=ON                   \
	-DENABLED_LOCAL_INFILE:BOOL=ON                \
	-DENABLED_THREAD_SAFE_CLIENT:BOOL=ON          \
	-DENABLED_EMBEDDED_SERVER:BOOL=OFF             \
	-DWITH_CLIENT_LDFLAGS:STRING=all-static                 \
	-DINSTALL_LAYOUT:STRING=STANDALONE            \
	-DCOMMUNITY_BUILD:BOOL=ON; 

这里需要注意，若多次运行cmake，系统将报如下错误：

	...
	CMake Error at cmake/readline.cmake:85 (MESSAGE):  Curses library not found.  Please install appropriate package,
	remove CMakeCache.txt and rerun cmake.On Debian/Ubuntu, package name is libncurses5-dev, on Redhat and derivates it is ncurses-devel.Call Stack (most recent call first):
	...

此时需要删除当前目录下CMakeCache.txt文件。

	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# rm -rf CMakeCache.txt

再次运行cmake命令即可。编译成功后，可以进行安装。

	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# make && make install

### 编译选项说明

指定安装文件的安装路径时常用的选项。

	-DCMAKE_INSTALL_PREFIX=/usr/local/mysql     #指定残可安装路径（默认的就是/usr/local/mysql）
	-DMYSQL_DATADIR=/data/mysql          #mysql的数据文件路径
	-DSYSCONFDIR=/etc                #配置文件路径

编译过程中启用其他存储引擎时指令介绍。

	-DWITH_INNOBASE_STORAGE_ENGINE=1         #使用INNOBASE存储引擎
	-DWITH_ARCHIVE_STORAGE_ENGINE=1            #常应用于日志记录和聚合分析，不支持索引
	-DWITH_BLACKHOLE_STORAGE_ENGINE=1      #黑洞存储引擎

编译过程中取消一些存储引擎指令介绍。

	-DWITHOUT_<ENGINE>_STORAGE_ENGINE=1
	#示例如下：
	-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1
	-DWITHOUT_FEDERATED_STORAGE_ENGINE=1
	-DWITHOUT_PARTITION_STORAGE_ENGINE=1

编译进过程中功能启用的指令介绍。

	-DWITH_READLINE=1       #支持批量导入mysql数据
	-DWITH_SSL=system       #mysql支持ssl会话，实现基于ssl的数据复
	-DWITH_ZLIB=system      #压缩库
	-DWITH_LIBWRAP=0        #是否可以基于WRAP实现访问控制

其他功能指令。

	-DMYSQL_TCP_PORT=3306                   #默认端口
	-DMYSQL_UNIX_ADDR=/tmp/mysql.sock       #默认套接字文件路径
	-DENABLED_LOCAL_INFILE=1                #是否启用LOCAL_INFILE功能
	-DEXTRA_CHARSETS=all  #是否支持额外的字符集
	-DDEFAULT_CHARSET=utf8                  #默认编码机制
	-DDEFAULT_COLLATION=utf8_general_ci     #设定默认语言的排序规则
	-DWITH_DEBUG=0                          #DEBUG功能设置
	-DENABLE_PROFILING=1                    #性能分析功能是否启用

关于编译选项说明，资料来源[CentOS6.4+MySQL-5.6.12安装详解](http://freeloda.blog.51cto.com/2033581/1252067)。

### 后续配置

复制配置文件my.cnf。

	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# cp support-files/my-large.cnf /etc/my.cnf

设置Mysql服务脚本。

	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# cp support-files/mysql.server /etc/init.d/mysqld #复制脚本
	[root@chenllcentos Percona-Server-5.5.34-rel32.0]# chmod +x /etc/init.d/mysqld #增加可执行权限
	[root@chenllcentos ~]# chkconfig --add mysqld #增加至sysV服务
	[root@chenllcentos ~]# chkconfig mysqld on  #开机自启动

初始化Mysql。

	[root@chenllcentos ~]# /usr/local/mysql/scripts/mysql_install_db --datadir=/usr/local/mysql/data --user=mysql

如果如下错误：

	FATAL ERROR: Could not find ./bin/my_print_defaults
	If you compiled from source, you need to run 'make install' to
	copy the software into the correct location ready for operation.
	If you are using a binary release, you must either be at the top
	level of the extracted archive, or pass the --basedir option
	pointing to that location.

请新增`--basedir`选项。

	[root@chenllcentos ~]# /usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --user=mysql
	
将Mysql命令添加到环境变量PATH。

	[root@chenllcentos ~]# echo 'export PATH=$PATH:/usr/local/mysql/bin' >> /etc/profile
	[root@chenllcentos ~]# source /etc/profile

启动Mysql服务。

	[root@chenllcentos ~]# service mysqld start
	Starting MySQL (Percona Server).. SUCCESS!

最后，配置Mysql用户密码。

	[root@chenllcentos ~]# mysqladmin -u root password 'yourpassword'

至此，编译安装Percona Server全部完成。接下来让我们看看，Transfer如何做到主从同步加速。  
下一篇，将介绍如何使用Transfer，实现Mysql主从同步加速。
