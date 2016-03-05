---
layout: post
title: "Linux运行Mysql多实例"
date: 2014-11-11 19:20:15 +0800
comments: true
tags: Database
keywords: linux, mysql, mysqld_multi
description: Linux运行Mysql多实例，Mysql不同版本多实例
---
## 引子

最近研究Mysql集群的问题，遇到一个需求，需要在同一台机器上运行多个Mysql实例。主要有两种实现方式，区别在于是否共享`my.cnf`配置文件。<!-- more -->  

- 分别配置`my.cnf`  

这种情况下，想要运行多个Mysql实例，就需要分别安装多个Mysql，无论是`socket`、`port`、`pid-file`、`datadir`还是`my.cnf`都是彼此独立的。具体实现较为简单，只需保证目录之间的差异性即可，但不利于后期数据库迁移。  

- 共用`my.cnf`  

如果采用共用`my.cnf`的方式实现Mysql多实例，需要在`my.cnf`额外配置[mysqld_multi]节点，该节点包含各个Mysql实例的连接信息和数据存储目录。  
考虑到方便统一部署和后期数据迁移，本文主要介绍第二种方案来实现Mysql多实例。
## 具体步骤
### 安装Mysql
建议最好采用编译安装，可进行最大程度的定制化，方便对Mysql进行配置。关于编译安装的详细步骤，可参考[编译安装Percona Server并使用Transfer实现主从数据库同步加速（上）](http://aaronchansunny.github.io/blog/2014/11/06/install-percona-server/)。
### 获取`mysqld_multi`配置样例
安装完Mysql后，可以运行如下命令，获取官方提供的Mysql多实例配置样例：

	[root@chenllcentos ~]# mysqld_multi --example
	# This is an example of a my.cnf file for mysqld_multi.
	# Usually this file is located in home dir ~/.my.cnf or /etc/my.cnf
	#
	# SOME IMPORTANT NOTES FOLLOW:
	#
	# 1.COMMON USER
	#
	#   Make sure that the MySQL user, who is stopping the mysqld services, has
	#   the same password to all MySQL servers being accessed by mysqld_multi.
	#   This user needs to have the 'Shutdown_priv' -privilege, but for security
	#   reasons should have no other privileges. It is advised that you create a
	#   common 'multi_admin' user for all MySQL servers being controlled by
	#   mysqld_multi. Here is an example how to do it:
	#
	#   GRANT SHUTDOWN ON *.* TO multi_admin@localhost IDENTIFIED BY 'password'
	#
	#   You will need to apply the above to all MySQL servers that are being
	#   controlled by mysqld_multi. 'multi_admin' will shutdown the servers
	#   using 'mysqladmin' -binary, when 'mysqld_multi stop' is being called.
	#
	# 2.PID-FILE
	#
	#   If you are using mysqld_safe to start mysqld, make sure that every
	#   MySQL server has a separate pid-file. In order to use mysqld_safe
	#   via mysqld_multi, you need to use two options:
	#
	#   mysqld=/path/to/mysqld_safe
	#   ledir=/path/to/mysqld-binary/
	#
	#   ledir (library executable directory), is an option that only mysqld_safe
	#   accepts, so you will get an error if you try to pass it to mysqld directly.
	#   For this reason you might want to use the above options within [mysqld#]
	#   group directly.
	#
	# 3.DATA DIRECTORY
	#
	#   It is NOT advised to run many MySQL servers within the same data directory.
	#   You can do so, but please make sure to understand and deal with the
	#   underlying caveats. In short they are:
	#   - Speed penalty
	#   - Risk of table/data corruption
	#   - Data synchronising problems between the running servers
	#   - Heavily media (disk) bound
	#   - Relies on the system (external) file locking
	#   - Is not applicable with all table types. (Such as InnoDB)
	#     Trying so will end up with undesirable results.
	#
	# 4.TCP/IP Port
	#
	#   Every server requires one and it must be unique.
	#
	# 5.[mysqld#] Groups
	#
	#   In the example below the first and the fifth mysqld group was
	#   intentionally left out. You may have 'gaps' in the config file. This
	#   gives you more flexibility.
	#
	# 6.MySQL Server User
	#
	#   You can pass the user=... option inside [mysqld#] groups. This
	#   can be very handy in some cases, but then you need to run mysqld_multi
	#   as UNIX root.
	#
	# 7.A Start-up Manage Script for mysqld_multi
	#
	#   In the recent MySQL distributions you can find a file called
	#   mysqld_multi.server.sh. It is a wrapper for mysqld_multi. This can
	#   be used to start and stop multiple servers during boot and shutdown.
	#
	#   You can place the file in /etc/init.d/mysqld_multi.server.sh and
	#   make the needed symbolic links to it from various run levels
	#   (as per Linux/Unix standard). You may even replace the
	#   /etc/init.d/mysql.server script with it.
	#
	#   Before using, you must create a my.cnf file either in /usr/local/mysql/my.cnf
	#   or /root/.my.cnf and add the [mysqld_multi] and [mysqld#] groups.
	#
	#   The script can be found from support-files/mysqld_multi.server.sh
	#   in MySQL distribution. (Verify the script before using)
	#

	[mysqld_multi]
	mysqld     = /usr/local/mysql/bin/mysqld_safe
	mysqladmin = /usr/local/mysql/bin/mysqladmin
	user       = multi_admin
	password   = my_password

	[mysqld2]
	socket     = /tmp/mysql.sock2
	port       = 3307
	pid-file   = /data/mysql2/hostname.pid2
	datadir    = /data/mysql2
	language   = /usr/local/mysql/share/mysql/english
	user       = unix_user1

	[mysqld3]
	mysqld     = /path/to/mysqld_safe
	ledir      = /path/to/mysqld-binary/
	mysqladmin = /path/to/mysqladmin
	socket     = /tmp/mysql.sock3
	port       = 3308
	pid-file   = /data/mysql3/hostname.pid3
	datadir    = /data/mysql3
	language   = /usr/local/mysql/share/mysql/swedish
	user       = unix_user2

`mysqld_multi --example`会自动读取本机Mysql的安装信息，包括`mysqld`和`mysqladmin`目录，并初始化`socket`、`portpid-file`和`datadir`信息。此处以运行两个Mysql实例为例子，修改[mysqld_multi]如下：

	[mysqld_multi]
	mysqld     = /usr/local/mysql/bin/mysqld_safe
	mysqladmin = /usr/local/mysql/bin/mysqladmin
	user       = root
	password   = root

	[mysqld2]
	socket     = /tmp/mysql.sock2
	port       = 3307
	pid-file   = /data/mysql2/chenllcentos.localdomain.pid2
	datadir    = /data/mysql2
	user       = mysql

	[mysqld3]
	socket     = /tmp/mysql.sock3
	port       = 3308
	pid-file   = /data/mysql3/chenllcentos.localdomain.pid3
	datadir    = /data/mysql3
	user       = mysql
### 添加多实例配置信息
打开`my.cnf`:

	[root@chenllcentos ~]# vi /etc/my.cnf

将[mysqld_multi]配置信息添加至`my.cnf`结尾。
### 配置数据目录
由[mysqld_multi]配置信息可以知道，我们对两个Mysql实例分别配置了`datadir`，该目录的设置非常重要，直接影响到多实例的启动是否成功。  
创建数据目录：

	[root@chenllcentos ~]# mkdir -p /data/mysql2
	[root@chenllcentos ~]# mkdir -p /data/mysql3

初始化数据库：

	[root@chenllcentos ~]# /usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/data/mysql2 --user=mysql
	[root@chenllcentos ~]# /usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/data/mysql3 --user=mysql

更改数据目录所属用户组：

	[root@chenllcentos ~]# chown -R mysql.mysql /data/mysql2
	[root@chenllcentos ~]# chown -R mysql.mysql /data/mysql3
### 启动和停止Mysql多实例
Mysql多实例的启动和停止使用`mysqld_multi`进行管理，例如需要同时启动mysqld2实例和mysqld3实例，可运行命令：

	[root@chenllcentos ~]# mysqld_multi start 2-3

若要停止数据库实例，命令类似：
	
	[root@chenllcentos ~]# mysqld_multi stop 2-3

查看数据库端口运行状态：

	[root@chenllcentos ~]# netstat -ntlp | grep mysqld 
	tcp        0      0 0.0.0.0:3307                0.0.0.0:*                   LISTEN      16975/mysqld        
	tcp        0      0 0.0.0.0:3308                0.0.0.0:*                   LISTEN      16976/mysqld

至此，同一主机下运行多个Mysql实例配置成功。
### 后续配置
为`mysqld2`分配用户：

	[root@chenllcentos ~]# mysqladmin -u root password 'root' -hlocalhost -P3307 -S /tmp/mysql.sock2

为`mysqld3`分配用户：

	[root@chenllcentos ~]# mysqladmin -u root password 'root' -hlocalhost -P3308 -S /tmp/mysql.sock3

这里需要注意，由于Mysql默认读取的socket文件是`/tmp/mysql.sock`，而在多实例环境下该文件是不存在的。因此，如果不通过`-S`指定socket文件的话，将报无法连接Mysql服务的错误：

	ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)

登录数据库，同样需要指定连接端口和socket文件。现以登录mysqld2为例：

	[root@chenllcentos ~]# mysql -uroot -proot -P3307 -S /tmp/mysql.sock2
	Welcome to the MySQL monitor.  Commands end with ; or \g.
	Your MySQL connection id is 1
	Server version: 5.5.34-log Source distribution

	Copyright (c) 2009-2013 Percona LLC and/or its affiliates
	Copyright (c) 2000, 2013, Oracle and/or its affiliates. All rights reserved.

	Oracle is a registered trademark of Oracle Corporation and/or its
	affiliates. Other names may be trademarks of their respective
	owners.

	Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

	mysql>
## Mysql不同版本多实例
以上介绍的是同一个Mysql版本单机启动多实例，那如果Mysql版本不同呢？回想下，在`my.cnf`配置文件[mysqld_multi]节点下并没有指定版本的信息入口。其实，只需要在启动时，指定不同的my.cnf即可。现在以同时启动Mysql5.1版本和编译安装的Percona Server版本多实例为例子，进行说明。  
首先，为了避免配置文件冲突，修改Percona Server配置文件名称：

	[root@chenllcentos ~]# mv /etc/my.cnf /etc/my.cnfp
	
接下来，`yum`安装Mysql，安装结果如下：

	[root@chenllcentos ~]# rpm -aq | grep mysql
	mysql-5.1.73-3.el6_5.x86_64
	mysql-server-5.1.73-3.el6_5.x86_64
	mysql-libs-5.1.73-3.el6_5.x86_64
	mysql-devel-5.1.73-3.el6_5.x86_64

可以知道，通过`yum`安装的Mysql版本为5.1。此时我们有两个Mysql配置文件：

	[root@chenllcentos ~]# ls -al /etc/my*
	-rw-r--r--. 1 root root  295 11月 12 10:41 /etc/my.cnf
	-rw-------. 1 root root 5076 11月 11 18:37 /etc/my.cnfp

通过Service启动Mysql5.1：

	[root@chenllcentos ~]# service mysqld start

通过mysqld_multi再启动Percona Server两个实例：

	[root@chenllcentos ~]# mysqld_multi --defaults-file=/etc/my.cnfp start 2-3

查看网络端口，验证是否启动成功：

	[root@chenllcentos ~]# netstat -ntlp | grep mysqld 
	tcp        0      0 0.0.0.0:3307                0.0.0.0:*                   LISTEN      20132/mysqld        
	tcp        0      0 0.0.0.0:3308                0.0.0.0:*                   LISTEN      20137/mysqld        
	tcp        0      0 0.0.0.0:3306                0.0.0.0:*                   LISTEN      18091/mysqld    

可以知道，目前有三个Mysql实例在运行。其中3306为Mysql5.1，3307和3308为Percona Server。  
通过`ps`命令查看进程：

	[root@chenllcentos ~]# ps -ef | grep mysql
	root     17983     1  0 Nov12 pts/0    00:00:00 /bin/sh /usr/bin/mysqld_safe --datadir=/var/lib/mysql --socket=/var/lib/mysql/mysql.sock --pid-file=/var/run/mysqld/mysqld.pid --basedir=/usr --user=mysql
	mysql    18091 17983  0 Nov12 pts/0    00:00:30 /usr/libexec/mysqld --basedir=/usr --datadir=/var/lib/mysql --user=mysql --log-error=/var/log/mysqld.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/var/lib/mysql/mysql.sock
	root     19678     1  0 11:08 pts/0    00:00:00 /bin/sh /usr/local/mysql/bin/mysqld_safe --socket=/tmp/mysql.sock2 --port=3307 --pid-file=/data/mysql2/hostname.pid2 --datadir=/data/mysql2 --user=mysql
	root     19685     1  0 11:08 pts/0    00:00:00 /bin/sh /usr/local/mysql/bin/mysqld_safe --socket=/tmp/mysql.sock3 --port=3308 --pid-file=/data/mysql3/hostname.pid3 --datadir=/data/mysql3 --user=mysql
	mysql    20132 19678  0 11:08 pts/0    00:00:00 /usr/local/mysql/bin/mysqld --basedir=/usr/local/mysql --datadir=/data/mysql2 --plugin-dir=/usr/local/mysql/lib/mysql/plugin --user=mysql --log-error=/var/log/mysqld.log --pid-file=/data/mysql2/hostname.pid2 --socket=/tmp/mysql.sock2 --port=3307
	mysql    20137 19685  0 11:08 pts/0    00:00:00 /usr/local/mysql/bin/mysqld --basedir=/usr/local/mysql --datadir=/data/mysql3 --plugin-dir=/usr/local/mysql/lib/mysql/plugin --user=mysql --log-error=/var/log/mysqld.log --pid-file=/data/mysql3/hostname.pid3 --socket=/tmp/mysql.sock3 --port=3308
	root     20217 22772  0 11:37 pts/0    00:00:00 grep mysql


Mysql不同版本多实例配置成功。