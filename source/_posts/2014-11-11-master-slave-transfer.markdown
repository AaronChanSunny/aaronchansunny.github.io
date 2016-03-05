---
layout: post
title: "编译安装Percona Server并使用Transfer实现主从数据库同步加速（下）"
date: 2014-11-11 09:39:51 +0800
comments: true
tags: Database
keywords: percona server, mysql, transfer
description: 编译安装Percona Server并使用Transfer实现主从数据库同步加速（下）
---
## 引子
MySQL的主从同步一直有从库延迟的问题，背景资料网上很多，原因简单描述如下：

1. MySQL从库上有一个IO线程负责从主库取binlog到写到本地。另外有一个SQL线程负责执行这些本地日志，实现命令重放；  

2. 正常网络状况下IO线程没有性能问题（这个待会会用到），问题是SQL线程只有一个，导致更新速度跟不上。所以经常会看到明明从库的CPU idle很高，但同步性能就是上不去。<!-- more -->  

既然主从同步的性能问题是由于SQL单线程导致的，最直接的想法就是把它改成多线程版本。Transfer就是这么一个工具，它在MySQL源码的基础上打patch，以多线程的方式实现主从数据同步，达到提高同步性能的目的。基本结构如下：  
![](/images/2014/11/01.jpg)  
以上内容参考[追风刀·丁奇](http://dinglin.iteye.com/)的博客。

### 安装与配置

目前，Transfer的最新版本是[Transfer 2.3](http://pan.baidu.com/s/1gdEdqTH)，基于版本Percona 5.5.34。安装方法非常简单，直接将下载的Transfer 2.3 替换官方版的mysqld，现在对Slave模式和Transfer模式分别进行说明。至于要如何安装Percona 5.5.34，可以参考[编译安装Percona Server并使用Transfer实现主从数据库同步加速（上）](http://aaronchansunny.github.io/blog/2014/11/06/install-percona-server/)。  

### Slave模式

Transfer的Slave模式跑起来比较简单，这里假设原生主从同步已经配置好，详细配置过程可以参考[Mysql主从复制和读写分离方案分析](http://aaronchansunny.github.io/blog/2014/11/05/mysql-read-write/)。然后，只需将Slave上的mysqld替换为Transfer，现在以本机配置为例进行说明。  
启动Mysql后，通过ps命令查看mysqld所在目录：

	[root@chenllcentos ~]# ps -ef | grep mysqld
	root     14156     1  0 Nov06 ?        00:00:00 /bin/sh /usr/local/mysql/bin/mysqld_safe --datadir=/usr/local/mysql/data --pid-file=/usr/local/mysql/data/chenllcentos.localdomain.pid
	mysql    14433 14156  0 Nov06 ?        00:04:49 /usr/local/mysql/bin/mysqld --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --plugin-dir=/usr/local/mysql/lib/mysql/plugin --user=mysql --log-error=/usr/local/mysql/data/chenllcentos.localdomain.err --pid-file=/usr/local/mysql/data/chenllcentos.localdomain.pid --socket=/tmp/mysql.sock --port=3306
	root     22894 22772  0 11:27 pts/0    00:00:00 grep mysqld

可以知道，当前mysqld所在目录为`/usr/local/mysql/bin`。  
备份mysqld为mysqld_bak：

	[root@chenllcentos ~]# mv /usr/local/mysql/bin/mysqld /usr/local/mysql/bin/mysqld_bak

复制Transfer到`/usr/local/mysql/bin`目录下：

	[root@chenllcentos ~]# cp ~/dev/transfer.2.3-based-PS-5.5.34 /usr/local/mysql/bin

重命名`transfer.2.3-based-PS-5.5.34`为`mysqld`:

	[root@chenllcentos ~]# mv /usr/local/mysql/bin/transfer.2.3-based-PS-5.5.34 /usr/local/mysql/bin/mysqld

添加mysqld可执行权限：

	[root@chenllcentos ~]# chmod 755 /usr/local/mysql/bin/mysqld

重启mysql服务：

	[root@chenllcentos ~]# service mysqld restart

到这里，我们就将原生的Slave数据库替换成Transfer 2.3 Slave啦。登录Mysql数据库，使用`mysql> show slave status\G`看看主从同步是否成功。

	mysql> show slave status\G
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 1XX.XX.XX.181
	                  Master_User: slave
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: mysql-bin.000019
	          Read_Master_Log_Pos: 192479
	               Relay_Log_File: chenllcentos-relay-bin.000030
	                Relay_Log_Pos: 252
	        Relay_Master_Log_File: mysql-bin.000019
	             Slave_IO_Running: Yes
	            Slave_SQL_Running: Yes
	              Replicate_Do_DB: 
	          Replicate_Ignore_DB: 
	           Replicate_Do_Table: 
	       Replicate_Ignore_Table: 
	      Replicate_Wild_Do_Table: 
	  Replicate_Wild_Ignore_Table: 
	                   Last_Errno: 0
	                   Last_Error: 
	                 Skip_Counter: 0
	          Exec_Master_Log_Pos: 192479
	              Relay_Log_Space: 415
	              Until_Condition: None
	               Until_Log_File: 
	                Until_Log_Pos: 0
	           Master_SSL_Allowed: No
	           Master_SSL_CA_File: 
	           Master_SSL_CA_Path: 
	              Master_SSL_Cert: 
	            Master_SSL_Cipher: 
	               Master_SSL_Key: 
	        Seconds_Behind_Master: 0
	Master_SSL_Verify_Server_Cert: No
	                Last_IO_Errno: 0
	                Last_IO_Error: 
	               Last_SQL_Errno: 0
	               Last_SQL_Error: 
	  Replicate_Ignore_Server_Ids: 
	             Master_Server_Id: 1

可以看出，此时`Slave_IO_Running`和`Slave_SQL_Running`都显示为`Yes`，因此Transfer Slave模式配置成功。

### Transfer模式

使用Transfer模式相比Slave模式复杂一些，因为需要多配置并维护一个MySQL实例。此时的Transfer其实就是一个运行的MySQL实例，配置成Master的从库，然后Slave再配置成Transfer的从库，当Transfer收到Master的Binlog后转发给后端多个Slave数据库，实现多线程数据同步。  

> 若没有多主的需求，测试数据库主从关系为Master->Transfer->Slave。

Transfer既可以部署在单独机器上，也可以部署在Slave机器上。其实两种部署方式在性能上并没有区别，只是若部署在同一机器上，需要配置MySQL多实例运行。作者推荐的线上配置是同Slave部署在同一机器。接下来，以同一机器部署为例进行说明。  

#### 原型环境

- 服务器A  

IP：1XX.XX.XX.181  
部署Master数据库  

- 服务器B  

IP：1XX.XX.XX.182  
部署Transfer和Slave数据库
#### 替换mysqld文件
将官方版的mysqld替换成下载的Transfer 2.3，这一步骤和Slave模式相同。
#### 修改配置文件
由于需要单机启动MySQL多实例，这里采用mysqld_multi对MySQL进行管理。  
复制`/etc/my.cnf`为`/etc/multi.cnf`，用于管理MySQL多实例：

	[root@chenllcentos ~]# cp /etc/my.cnf /etc/multi.cnf

打开`/etc/multi.cnf`：

	[root@chenllcentos ~]# vi /etc/multi.cnf

在文件末尾添加如下内容：

	[mysqld_multi]
	mysqld     = /usr/local/mysql5/bin/mysqld_safe
	mysqladmin = /usr/local/mysql5/bin/mysqladmin
	user       = root
	password   = root

	[mysqld2]
	socket     = /tmp/mysql.sock2
	port       = 3306
	pid-file   = /data/mysql5/data2/hostname.pid2
	datadir    = /data/mysql5/data2
	user       = mysql

	server-id=18255
	binlog-do-db=test
	log-bin=mysql-bin

	transfer_mode = on

	remote_slave_hostname = 1XX.XX.XX.182
	remote_slave_username = root
	remote_slave_password = root
	remote_slave_port = 13306

其中`server-id`、`binlog-do-db`和`log-bin`是传统主从配置，`transfer_mode`和`remote_slave_*`是Transfer模式配置。

> Transfer参数说明：  
  transfer_parallel_on  
  1.on—多线程复制  
  2.off—单线程 默认值on  
  transfer_mode  
  1.on—多线程复制  
  2.off—单线程 默认值on  
  remote_slave_*  
  具有访问Slave的super权限帐号
  
使用`mysqld_multi`启动Transfer，并指定配置文件为：`/etc/multi.cnf`。

	[root@chenllcentos ~]# mysqld_multi --defaults-file=/etc/multi.cnf start 2

登录Transfer：

	[root@chenllcentos ~]# mysql -uroot -proot -S /tmp/mysql.sock2 

> 这里需要指定MySQL连接为：`/tmp/mysql.sock2`。

查看刚才配置的Transfer参数：

	mysql> show variables like "transfer%";
	+-------------------------+---------------+
	| Variable_name           | Value         |
	+-------------------------+---------------+
	| transfer_mode           | ON            |
	| transfer_parallel_on    | ON            |
	| transfer_slave_host     | 1XX.XX.XX.182 |
	| transfer_slave_password | ****          |
	| transfer_slave_port     | 13306         |
	| transfer_slave_thread   | 16            |
	| transfer_slave_username | root          |
	| transfer_verbos         | OFF           |
	+-------------------------+---------------+
	8 rows in set (0.00 sec)

#### 配置Slave

之前已经配置好了Transfer，现在需要配置Slave真实数据库。在Master->Transfer->Slave模式下，只需要保证Master和Slave的MySQL版本一致即可。因此，本例子中Slave使用的是MySQL5.1版本，采用`yum`安装方式部署。修改配置信息如Transfer指定的`transfer_slave_*`参数信息。`/etc/my.cnf`配置文件内容如下：

	[mysqld]
	port = 13306

	server-id = 18251

	datadir=/var/lib/mysql
	socket=/var/lib/mysql/mysql.sock
	user=mysql
	# Disabling symbolic-links is recommended to prevent assorted security risks
	symbolic-links=0

	[mysqld_safe]
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid

配置Slave账户信息：

	[root@chenllcentos ~]# mysqladmin -u root password 'root'

启动Slave：

	[root@chenllcentos ~]# service mysqld start

若启动失败，请关闭SELinux：

	[root@chenllcentos ~]# setenforce 0
	[root@chenllcentos ~]# getenforce
	Permissive

#### 同步表结构

在Transfer模式下，必须保证Master->Transfer->Slave需要同步的表结构相同。其中Transfer只需要同步表结构即可，不需要数据，这是因为Transfer在主从同步过程中只负责转发Binlog，并不保存任何数据。由于例子中设置主从同步test数据库，因此需要同步test表结构。  
dump主库test表结构：

	[root@chenllcentos ~]# mysqldump -uroot -proot --no-data --add-drop-table --add-drop-database test > /data/db.sql

分别登录Transfer和Slave数据库，执行`db.sql`脚本文件：

	mysql> use test
	mysql> source /data/db.sql

#### Transfer模式主从配置

在Transfer模式下，若无多主的需求，Transfer设置为Master的从库，然后Slave设置成Transfer的从库，具体的配置流程和传统主从配置相。至此，Transfer模式配置完成，现在让我们一起验证。
#### Transfer模式验证
首先，登录服务器B Transfer数据库：

	[root@chenllcentos ~]# mysql -uroot -proot -S /tmp/mysql.sock2

查询Slave状态：

	mysql> show slave status\G
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 1XX.XX.XX.181
	                  Master_User: transfer
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: mysql-bin.000021
	          Read_Master_Log_Pos: 3148
	               Relay_Log_File: hostname-relay-bin.000070
	                Relay_Log_Pos: 440
	        Relay_Master_Log_File: mysql-bin.000021
	             Slave_IO_Running: Yes
	            Slave_SQL_Running: Yes
	              Replicate_Do_DB: 
	          Replicate_Ignore_DB: 
	           Replicate_Do_Table: 
	       Replicate_Ignore_Table: 
	      Replicate_Wild_Do_Table: 
	  Replicate_Wild_Ignore_Table: 
	                   Last_Errno: 0
	                   Last_Error: 
	                 Skip_Counter: 0
	          Exec_Master_Log_Pos: 3148
	              Relay_Log_Space: 744
	              Until_Condition: None
	               Until_Log_File: 
	                Until_Log_Pos: 0
	           Master_SSL_Allowed: No
	           Master_SSL_CA_File: 
	           Master_SSL_CA_Path: 
	              Master_SSL_Cert: 
	            Master_SSL_Cipher: 
	               Master_SSL_Key: 
	        Seconds_Behind_Master: 0
	Master_SSL_Verify_Server_Cert: No
	                Last_IO_Errno: 0
	                Last_IO_Error: 
	               Last_SQL_Errno: 0
	               Last_SQL_Error: 
	  Replicate_Ignore_Server_Ids: 
	             Master_Server_Id: 1
	1 row in set (0.00 sec)

可以知道，Transfer的主库设置为服务器A的Master数据库。
再登录服务器B Slave数据库：

	[root@chenllcentos ~]# mysql -uroot -proot 

查看Slave状态：

	mysql> show slave status\G
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 1XX.XX.XX.182
	                  Master_User: slave2
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: mysql-bin.000022
	          Read_Master_Log_Pos: 503
	               Relay_Log_File: mysqld-relay-bin.000036
	                Relay_Log_Pos: 252
	        Relay_Master_Log_File: mysql-bin.000022
	             Slave_IO_Running: Yes
	            Slave_SQL_Running: Yes
	              Replicate_Do_DB: 
	          Replicate_Ignore_DB: 
	           Replicate_Do_Table: 
	       Replicate_Ignore_Table: 
	      Replicate_Wild_Do_Table: 
	  Replicate_Wild_Ignore_Table: 
	                   Last_Errno: 0
	                   Last_Error: 
	                 Skip_Counter: 0
	          Exec_Master_Log_Pos: 503
	              Relay_Log_Space: 554
	              Until_Condition: None
	               Until_Log_File: 
	                Until_Log_Pos: 0
	           Master_SSL_Allowed: No
	           Master_SSL_CA_File: 
	           Master_SSL_CA_Path: 
	              Master_SSL_Cert: 
	            Master_SSL_Cipher: 
	               Master_SSL_Key: 
	        Seconds_Behind_Master: 0
	Master_SSL_Verify_Server_Cert: No
	                Last_IO_Errno: 0
	                Last_IO_Error: 
	               Last_SQL_Errno: 0
	               Last_SQL_Error: 
	1 row in set (0.00 sec)

这里，Slave的主库设置成同台主机下的另一个MySQL实例，即Transfer。  
现在，我们在Master插入一条数据，看看主从是否生效：  

	mysql> insert into test.t_user (user_id, user_name, user_address) values (3, 'Mike', 'Fuzhou');
	Query OK, 1 row affected (0.01 sec)

查询Transfer test数据库t_user表：

	mysql> select * from test.t_user;
	Empty set (0.00 sec)

这里，为什么是空的呢？大家还记得吗，前面提到过，在Transfer模式下，Transfer只负责转发Binlog，并不会保存任何数据。所以，在Transfer查询t_user表数据当然为空啦！  
接下来，让我们看看Slave数据：

	mysql> select * from test.t_user;
	+---------+-----------+--------------+
	| user_id | user_name | user_address |
	+---------+-----------+--------------+
	|       3 | Mike      | Fuzhou       |
	+---------+-----------+--------------+
	1 row in set (0.00 sec)

果然不出所料，刚才在Master插入的数据，现在已经同步到Slave上了。说明我们配置的Transfer模式已经成功运行。

### Transfer模式转换

Transfer模式已经部署运行，但是现在需求变化了，并不需要另外维护一个MySQL实例，而是想切换成Slave模式。Transfer提供了非常简单的实现，只需要更改配置文件参数即可，然后重启Transfer。若此时在Master t_user表插入记录，Transfer将直接进行复制，在这种情况下和传统的MySQL主从没有任何区别。让我们做个例子验证下。  
修改Transfer的`transfer_mode`参数：

	[root@chenllcentos ~]# vi /etc/multi.cnf

查找`transfer_mode`将值设置成`off`。  
重启Transfer：

	[root@chenllcentos ~]# mysqld_multi --defaults-file=/etc/multi.cnf stop 2
	[root@chenllcentos ~]# mysqld_multi --defaults-file=/etc/multi.cnf start 2

登录Transfer：

	[root@chenllcentos ~]# mysql -uroot -proot -S /tmp/mysql.sock2

查看Transfer参数：

		mysql> show variables like "transfer%";
	+-------------------------+---------------+
	| Variable_name           | Value         |
	+-------------------------+---------------+
	| transfer_mode           | OFF           |
	| transfer_parallel_on    | ON            |
	| transfer_slave_host     | 1XX.XX.XX.182 |
	| transfer_slave_password | ****          |
	| transfer_slave_port     | 13306         |
	| transfer_slave_thread   | 16            |
	| transfer_slave_username | root          |
	| transfer_verbos         | OFF           |
	+-------------------------+---------------+
	8 rows in set (0.00 sec)

登录Master数据库：

	[root@chenllcentos ~]# mysql -uroot -proot

往t_user表再插入一条数据：

	mysql> insert into test.t_user (user_id, user_name, user_address) values (1, 'Lucy', 'Xiamen');
	Query OK, 1 row affected (0.01 sec)

在Transfer中查询t_user表数据：

	mysql> select * from test.t_user;
	+---------+-----------+--------------+
	| user_id | user_name | user_address |
	+---------+-----------+--------------+
	|       1 | Lucy      | Xiamen       |
	+---------+-----------+--------------+
	1 row in set (0.00 sec)

可以发现，此时Transfer会同步Master刚插入的数据。那么，之前配置的Slave数据库呢？让我们看看。  
在Slave中查询t_user表数据：

	mysql> select * from test.t_user;
	+---------+-----------+--------------+
	| user_id | user_name | user_address |
	+---------+-----------+--------------+
	|       3 | Mike      | Fuzhou       |
	+---------+-----------+--------------+
	1 row in set (0.00 sec)

果然，Slave数据库并没有同步刚才在Master插入的`Lucy`记录。说明Transfer模式切换Slave模式成功。

## 小结

其实，在做Transfer模式的时候真正纠结了一把。多实例场景下，Transfer数据库的端口必须是3306。若更改Transfer端口为其他数值（例如3307），Slave始终无法连接上Transfer，报2013错误：

	[ERROR] Slave I/O: error connecting to master 'slave2@1XX.XX.XX.182:3307' - retry-time: 60  retries: 86400, Error_code: 2013

目前还未找到解决办法。无奈之下，只能修改Slave端口达到单主机配置Slave和Transfer的目的。
