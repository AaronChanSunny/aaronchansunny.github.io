---
layout: post
title: "Mysql主从复制和读写分离方案分析"
date: 2014-11-05 15:57:29 +0800
comments: true
tags: Database
keywords: mysql, read, write, amoeba
description: Mysql主从复制和读写分离方案分析
---

## 引子
最近在研究Web服务端负载均衡方面的技术，参考网上资料，总体思路可以分为如下几类：  
- 应用服务器集群，典型的代表就是Nginx+Tomcat实现负载均衡；  
- 数据库集群。   

本文主要关注数据库集群。  

## 实现思路

- 应用层解决方案  

通过应用层对数据源做路由来实现读写分离，项目是SpringMVC+myBatis，SQL路由交给Spring，通过AOP或者Annotation由代码显示的控制Datasource。  
优点是路由策略的扩展性和可控性较强。  
缺点是耦合到Spring；需要加入控制代码。  

- 中间件解决方案  

通过mysql中间件做主从集群，Mysql Proxy、Amoeba、Atlas等中间件貌似都能符合需求。  
优点是与应用层解耦。  
缺点是增加一个服务维护的风险点，性能及稳定性待测试，需要支持代码强制主从和事务。  
- 驱动解决方案  

Mysql自带的ReplicationDriver提供主从库访问的驱动，是通过保持多个数据源的链接并根据ReadOnly True/False来选择数据源。相当于应用层解决方案的一个现有实现，扩展性更弱。并且貌似不能使用其他驱动。由于耦合较高暂不考虑。 

### 三种实现思路关键技术

- 在应用层使用Spring对数据源做路由，关键字：Spring AOP；  
- 增加中间代理层，Amoeba就属于这种情况，此外还有Mysql官方提供的Mysql Proxy；  
- 在驱动层使用Mysql提供的主从库访问驱动，直接与数据库连接驱动耦合，扩展性弱，目前还未做原型尝试。  

综合上述分析，考虑到需要与应用层解耦，现采用中间件解决方案，使用Amoeba做SQL路由，实现数据库读写分离。  
既然选择使用Amoeba，让我们先了解什么是Amoeba？它能做什么？要怎么做？最后再看看它不能做什么。<!-- more -->

## Amoeba

### Amoeba是什么

Amoeba(变形虫)项目,该开源框架于2008年开始发布一款Amoeba for Mysql软件。详细资料可参阅[Amoeba](http://docs.hexnova.com/amoeba/what-is.html)官方文档(需翻墙)。

### Amoeba能做什么
Amoeba致力于MySQL的分布式数据库前端代理层，它主要在应用层访问MySQL的时候充当SQL路由功能，专注于分布式数据库代理层（Database Proxy）开发。座落与 Client、DB Server(s)之间，对客户端透明。具有负载均衡、高可用性、SQL过滤、读写分离、可路由相关的到目标数据库、可并发请求多台数据库合并结果。 通过Amoeba你能够完成多数据源的高可用、负载均衡、数据切片的功能。

### Amoeba不能做什么

既然知道Amoeba能为我们解决什么问题，也要做到Amoeba不擅长的事情。这样在具体项目技术方案选择时，方能权衡考虑。Amoeba对于以下几点暂时无能为力：   

- 目前还不支持事务；  
- 暂时不支持存储过程，官方说近期会支持；  
- 不适合从Amoeba导数据的场景或者对大数据量查询的query并不合适，比如一次请求返回10w以上甚至更多数据的场合；  
- 暂时不支持分库分表，amoeba目前只做到分数据库实例，每个被切分的节点需要保持库表结构一致。  
若实际项目中所需要的功能正式Amoeba的短板，建议使用Mysql Proxy作为中间件，或者在应用层通过程序控制数据源，手动实现数据库读写分离。

## 原型环境

- 服务器A  

IP: 1XX.XX.XX.181  
运行Mysql主数据库和Amoeba。  

- 服务器B  

IP: 1XX.XX.XX.182  
运行Mysql从数据库。  

- 服务器C  

IP: 1XX.XX.XX.183   
运行Mysql从数据库。  
OS版本。

	[root@chenllcentos ~]# cat /etc/redhat-release 
	CentOS release 6.5 (Final)

## 具体实现

Mysql数据库读写分离的具体实现主要包括两个部分配置，即数据主从复制和Amoeba代理，现分别进行介绍。

### 主从复制

为什么要进行主从复制呢，其实很容易理解，因为数据要同步啊。  
查看服务器A是否已经安装Mysql数据库。

	[root@chenllcentos ~]# rpm -aq | grep mysql

若无消息显示，则进行Mysql安装，否则跳过此步骤。

	yum install -y mysql-server mysql mysql-devel mysql-libs

Mysql安装完毕，默认开机不启动Mysql服务。

	[root@chenllcentos ~]# chkconfig --list | grep mysqld
	mysqld         	0:关闭	1:关闭	2:关闭	3:关闭	4:关闭	5:关闭	6:关闭

现在我们更改下配置，让Mysql开机启动。

	[root@chenllcentos ~]# chkconfig mysqld on
	[root@chenllcentos ~]# chkconfig --list | grep mysqld
	mysqld         	0:关闭	1:关闭	2:启用	3:启用	4:启用	5:启用	6:关闭

接下来，设置Mysql账户密码。  

  	[root@chenllcentos ~]# mysqladmin -u root password 'yourpassword'

此时,可以用刚才设置的账户密码登陆数据库。

	[root@chenllcentos ~]# mysql -uroot -pyourpassword

至此，Mysql数据库安装成功。同样的，对服务器B和服务器C安装Mysql数据库，此处略去。接下来，开始进行数据库主从复制的配置。  

- 主数据库配置  

修改主数据库配置文件my.cnf。

	[root@chenllcentos ~]# vi /etc/my.cnf

新增如下标注内容：

	[mysqld]
	max_connections=1000

	binlog-ignore-db=mysql #新增
	binlog-ignore-db=information_schema #新增

	log-bin=mysql-bin #新增
	server-id=1 #新增

	datadir=/var/lib/mysql
	socket=/var/lib/mysql/mysql.sock
	user=mysql
	# Disabling symbolic-links is recommended to prevent assorted security risks
	symbolic-links=0

	[mysqld_safe]
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid

关于新增的几项配置，有什么作用呢？其中binlog-ignore-db用来指定忽略同步的数据库，未指定的默认都进行主从复制。log-bin指定数据库操作日志，主从复制的过程本质就是从数据库在主数据库读取该日志文件，并且再执行一次。server-id只要满足在数据库集群中不重复即可。  
保存退出，重启Mysqld服务，使配置生效。额外提个原则，凡是修改到配置文件，最好都重启该配置相关的程序或服务。  

	[root@chenllcentos ~]# service mysqld restart
	停止 mysqld：                                              [确定]
	正在启动 mysqld：                                          [确定]

登陆主数据库。

	[root@chenllcentos ~]# mysql -uroot -pyourpassword

查看主数据库master状态。

	mysql> show master status\G
	*************************** 1. row ***************************
	            File: mysql-bin.000015
	        Position: 106
	    Binlog_Do_DB: 
	Binlog_Ignore_DB: mysql,information_schema
	
可以看出，Binlog_Ignore_DB显示的信息就是刚才我们在配置文件所配置的信息。此外，还有两个重要的参数需要记下：mysql-bin.000015和106。从数据库就是根据这两个参数，完成主从复制，以达到数据同步的效果。  
从数据库要读取主数据库日志文件，需要主数据开放授权用户。

	mysql> GRANT REPLICATION SLAVE ON *.* to 'slave'@'1XX.XX.XX.182' identified by 'root'
	mysql> GRANT REPLICATION SLAVE ON *.* to 'slave1'@'1XX.XX.XX.183' identified by 'root'

进行从数据库配置时，将使用到这两个授权用户。

出于数据安全性考虑，Mysql提供访问权限控制，若以主机的方式远程访问数据库，需要开启相应权限。

	mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'1XX.XX.XX.181' IDENTIFIED BY 'root' WITH GRANT OPTION;
	mysql> FLUSH PRIVILEGES;
	mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'1XX.XX.XX.182' IDENTIFIED BY 'root' WITH GRANT OPTION;
	mysql> FLUSH PRIVILEGES;
	mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'1XX.XX.XX.183' IDENTIFIED BY 'root' WITH GRANT OPTION;
	mysql> FLUSH PRIVILEGES;

最后，还需要修改iptables，对数据库端口3306放行。

	[root@chenllcentos ~]# vi /etc/sysconfig/iptables

新增如下语句：

	# 放行Mysql端口
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT

至此，完成主数据库配置。接下来，让我们进行从数据库配置。  

- 从数据库配置  

从数据库配置相对主数据配置相对简单，主要包括配置文件修改和主从复制设置。现以服务器B为例进行说明。   
修改从数据库配置文件。

	[mysqld]
	max_connections=1000

	server-id=2 #新增

	datadir=/var/lib/mysql
	socket=/var/lib/mysql/mysql.sock
	user=mysql
	# Disabling symbolic-links is recommended to prevent assorted security risks
	symbolic-links=0

	[mysqld_safe]
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid

设置主从数据库同步点。

	mysql> change master to master_host='1XX.XX.XX.181',master_user='slave',master_password='root',master_log_file='mysql-bin.000015',master_log_pos=106;

还记得mysql-bin.000015和106这两个参数吗？没错，就是我们在主数据库查看master状态所显示的信息。  
启动主从复制。

	mysql> slave start;

查询slave状态。

	mysql> show slave status\G
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 1XX.XX.XX.181
	                  Master_User: slave
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: mysql-bin.000015
	          Read_Master_Log_Pos: 106
	               Relay_Log_File: mysqld-relay-bin.000005
	                Relay_Log_Pos: 251
	        Relay_Master_Log_File: mysql-bin.000015
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
	          Exec_Master_Log_Pos: 106
	              Relay_Log_Space: 758
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

只有当Slave_IO_Running和Slave_SQL_Running都显示Yes时，才表示主从复制配置成功。否则失败，检查上述配置过程。  
服务器C从数据库的配置过程类似，此处略去。

### 主从复制验证

首先，在主数据建立一个demo数据库，看两个从数据库是否会自动进行复制。  
在服务器A登录主数据库，查看现有数据库。

	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| test               |
	+--------------------+

现在，新增一个测试数据库demo。

	mysql> create database demo;

接下来，分别登录服务器B和服务器C的从数据库，查询数据库。

	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| demo               |
	| mysql              |
	| test               |
	+--------------------+

可以发现，当主数据库发生改动，从数据库会相应同步，并且同步的过程是异步进行的。因此，可以验证我们配置的主从复制已经生效。

### Amoeba数据库代理

Amoeba作为数据库代理，以中间件的形式存在，拓扑图如下所示：

![](/images/2014/11/amoeba-mysql.png)
图片来源于Amoeba官网。

目前Amoeba for Mysql最新版本为[amoeba-mysql-3.0.5-RC-distribution.zip](http://sourceforge.net/projects/amoeba/files/)。  
安装过程很简单，只需要将zip压缩包解压至/usr/local/即可。若没有安装zip和unzip，可以通过centOS yum安装。

	[root@chenllcentos ~]# yum -y install zip unzip

接下来，解压Amoeba压缩包。

	[root@chenllcentos ~]# unzip amoeba-mysql-3.0.5-RC-distribution.zip
	[root@chenllcentos ~]# cp -rf amoeba-mysql-3.0.5-RC /usr/local

启动Amoeba。

	[root@chenllcentos ~]# /usr/local/amoeba-mysql-3.0.5-RC/bin/launcher

但是提示出现fatal exception：

	The stack size specified is too small, Specify at least 228k
	Error: Could not create the Java Virtual Machine.
	Error: A fatal exception has occurred. Program will exit.

从错误文字上看，应该是由于stack size太小，导致JVM启动失败，要如何修改呢？  
其实Amoeba已经考虑到这个问题，并将JVM参数配置写在属性文件里。现在，让我们通过该属性文件修改JVM参数。  
修改jvm.properties文件JVM_OPTIONS参数。

	[root@chenllcentos ~]# vi /usr/local/amoeba-mysql-3.0.5-RC/jvm.properties

将内容：

	JVM_OPTIONS="-server -Xms256m -Xmx1024m -Xss196k -XX:PermSize=16m -XX:MaxPermSize=96m"

替换为：

	JVM_OPTIONS="-server -Xms1024m -Xmx1024m -Xss256k -XX:PermSize=16m -XX:MaxPermSize=96m"

再次启动Amoeba。

	[root@chenllcentos ~]# /usr/local/amoeba-mysql-3.0.5-RC/bin/launcher

若使用Amoeba完成读写分离，需要分别对dbServers.xml和amoeba.xml两个配置文件进行配置。与在应用层实现读写分离不同，使用Amoeba实现读写分离只需要修改配置文件，并不会产生硬编码耦合，有利于系统扩展和维护。  

首先是配置dbServers.xml，主要是配置真实Mysql数据库连接信息。  

```
<?xml version="1.0" encoding="gbk"?>
<!DOCTYPE amoeba:dbServers SYSTEM "dbserver.dtd">
<amoeba:dbServers xmlns:amoeba="http://amoeba.meidusa.com/">

        <!-- 
            Each dbServer needs to be configured into a Pool,
            If you need to configure multiple dbServer with load balancing that can be simplified by the following configuration:
             add attribute with name virtual = "true" in dbServer, but the configuration does not allow the element with name factoryConfig
             such as 'multiPool' dbServer   
        -->

    <!-- 该dbServer节点abstractive="true"，包含Mysql的公共配置信息，其他dbServer节点都继承该节点 -->
    <!-- 设置节点配置的继承结构，可以避免重复配置相同信息，减少配置文件冗余 -->
    <dbServer name="abstractServer" abstractive="true">
        <factoryConfig class="com.meidusa.amoeba.mysql.net.MysqlServerConnectionFactory">
            <property name="connectionManager">${defaultManager}</property>
            <property name="sendBufferSize">64</property>
            <property name="receiveBufferSize">128</property>

            <!-- mysql port -->
            <!-- Mysql默认端口 -->
            <property name="port">3306</property>

            <!-- mysql schema -->
            <!-- 默认连接的数据库，若不存在需要事先创建，否则Amoeba启动报错 -->
            <property name="schema">test</property>

            <!-- mysql user -->
            <property name="user">root</property>

            <property name="password">root</property>
        </factoryConfig>

        <poolConfig class="com.meidusa.toolkit.common.poolable.PoolableObjectPool">
            <property name="maxActive">500</property>
            <property name="maxIdle">500</property>
            <property name="minIdle">1</property>
            <property name="minEvictableIdleTimeMillis">600000</property>
            <property name="timeBetweenEvictionRunsMillis">600000</property>
            <property name="testOnBorrow">true</property>
            <property name="testOnReturn">true</property>
            <property name="testWhileIdle">true</property>
        </poolConfig>
    </dbServer>

    <!-- master节点继承abstractServer -->
    <dbServer name="master"  parent="abstractServer">
        <factoryConfig>
            <!-- mysql ip -->
            <!-- master数据库主机地址 -->
            <property name="ipAddress">1XX.XX.XX.181</property>
        </factoryConfig>
    </dbServer>

    <!-- slave节点继承abstractServer -->
    <dbServer name="slave"  parent="abstractServer">
        <factoryConfig>
            <!-- mysql ip -->
            <!-- slave数据库主机地址 -->
            <property name="ipAddress">1XX.XX.XX.182</property>
        </factoryConfig>
    </dbServer>

    <!-- slave1节点继承abstractServer -->
    <dbServer name="slave1"  parent="abstractServer">
            <factoryConfig>
                    <!-- mysql ip -->
                    <!-- slave1数据库主机地址 -->
                    <property name="ipAddress">1XX.XX.XX.183</property>
            </factoryConfig>
        </dbServer>

   <dbServer name="server1"  parent="abstractServer">
            <factoryConfig>
                    <!-- mysql ip -->
                    <property name="ipAddress">1XX.XX.XX.181</property>
            </factoryConfig>
        </dbServer>

<dbServer name="server2"  parent="abstractServer">
            <factoryConfig>
                    <!-- mysql ip -->
                    <!-- server2数据库主机地址 -->
                    <property name="ipAddress">1XX.XX.XX.185</property>
            </factoryConfig>
        </dbServer>

    <!-- 配置数据库读取连接池 -->
    <dbServer name="readPool" virtual="true">
        <poolConfig class="com.meidusa.amoeba.server.MultipleServerPool">
            <!-- Load balancing strategy: 1=ROUNDROBIN , 2=WEIGHTBASED , 3=HA-->
            <property name="loadbalance">1</property>

            <!-- Separated by commas,such as: server1,server2,server1 -->
            <property name="poolNames">slave,slave1</property>
        </poolConfig>
    </dbServer>

</amoeba:dbServers>
```

可以看出，对dbServers.xml文件的配置，主要就是对dbServer节点的配置。其中，readPool节点需要特别注意，因为Amoeba实现读写分离就是根据它来实现。  
接下来是amoeba.xml，主要是配置代理数据库连接信息。

```
	<?xml version="1.0" encoding="gbk"?>

	<!DOCTYPE amoeba:configuration SYSTEM "amoeba.dtd">
	<amoeba:configuration xmlns:amoeba="http://amoeba.meidusa.com/">

		<proxy>
		
			<!-- service class must implements com.meidusa.amoeba.service.Service -->
			<service name="Amoeba for Mysql" class="com.meidusa.amoeba.mysql.server.MySQLService">
				<!-- port -->
				<property name="port">8066</property>
				
				<!-- bind ipAddress -->
				<!-- 
				<property name="ipAddress">1XX.XX.XX.181</property>
				 -->
				
				<property name="connectionFactory">
					<bean class="com.meidusa.amoeba.mysql.net.MysqlClientConnectionFactory">
						<property name="sendBufferSize">128</property>
						<property name="receiveBufferSize">64</property>
					</bean>
				</property>
				
				<property name="authenticateProvider">
					<bean class="com.meidusa.amoeba.mysql.server.MysqlClientAuthenticator">
						
						<property name="user">root</property>
						
						<property name="password">aroot</property>
						
						<property name="filter">
							<bean class="com.meidusa.toolkit.net.authenticate.server.IPAccessController">
								<property name="ipFile">${amoeba.home}/conf/access_list.conf</property>
							</bean>
						</property>
					</bean>
				</property>
				
			</service>
			
			<runtime class="com.meidusa.amoeba.mysql.context.MysqlRuntimeContext">
				
				<!-- proxy server client process thread size -->
				<property name="executeThreadSize">128</property>
				
				<!-- per connection cache prepared statement size  -->
				<property name="statementCacheSize">500</property>
				
				<!-- default charset -->
				<property name="serverCharset">utf8</property>
				
				<!-- query timeout( default: 60 second , TimeUnit:second) -->
				<property name="queryTimeout">60</property>
			</runtime>
			
		</proxy>
		
		<!-- 
			Each ConnectionManager will start as thread
			manager responsible for the Connection IO read , Death Detection
		-->
		<connectionManagerList>
			<connectionManager name="defaultManager" class="com.meidusa.toolkit.net.MultiConnectionManagerWrapper">
				<property name="subManagerClassName">com.meidusa.toolkit.net.AuthingableConnectionManager</property>
			</connectionManager>
		</connectionManagerList>
		
			<!-- default using file loader -->
		<dbServerLoader class="com.meidusa.amoeba.context.DBServerConfigFileLoader">
			<property name="configFile">${amoeba.home}/conf/dbServers.xml</property>
		</dbServerLoader>
		
		<queryRouter class="com.meidusa.amoeba.mysql.parser.MysqlQueryRouter">
			<property name="ruleLoader">
				<bean class="com.meidusa.amoeba.route.TableRuleFileLoader">
					<property name="ruleFile">${amoeba.home}/conf/rule.xml</property>
					<property name="functionFile">${amoeba.home}/conf/ruleFunctionMap.xml</property>
				</bean>
			</property>
			<property name="sqlFunctionFile">${amoeba.home}/conf/functionMap.xml</property>
			<property name="LRUMapSize">1500</property>
			<property name="defaultPool">master</property>
			
			<property name="writePool">master</property>
			<property name="readPool">readPool</property>
		
			<property name="needParse">true</property>
		</queryRouter>
	</amoeba:configuration>
```

在amoeba.xml中，主要完成连接信息和SQL路由配置。在queryRouter节点中，通过配置writePool和readPool可以实现读写分离。  
配置完成后，重启Amoeba。

	[root@chenllcentos ~]# /usr/local/amoeba-mysql-3.0.5-RC/bin/shutdown
	[root@chenllcentos ~]# /usr/local/amoeba-mysql-3.0.5-RC/bin/launcher

至此，Mysql主从复制和使用Amoeba实现数据库读写分离全部配置完成。  

### 读写分离验证

接下来，进行简单测试，验证以上配置是否能够正确运行。  
登录master主数据库。

	[root@chenllcentos ~]# mysql -uroot -pyourpassword -h1XX.XX.XX.181 -P8066

额外说明下，此处的yourpassword是连接Amoeba的密码，也就是在amoeba.xml配置文件中配置的密码，与Mysql密码不同，需要注意。  
登陆后，此时会提示以下信息。  

	Server version: 5.1.45-mysql-amoeba-proxy-3.0.4-BETA Source distribution

说明已经成功连接Mysql代理Amoeba。  
为了验证Amoeba读写分离配置是否生效，我们做一个简单的测试。  
先在181服务器master服务器上创建一个表。

	mysql> create table sxit (id int(10) ,name varchar(10));

而后，分别停止服务器B和服务器C两个从数据库的主从复制，便于数据库操作观察。  
登陆服务器B从数据库。

	[root@chenllcentos ~]# mysql -uroot -pyourpassword

停止从数据库主从复制。

	mysql> slave stop;

登陆服务器C从数据库。

	[root@chenllcentos ~]# mysql -uroot -pyourpassword

停止从数据库主从复制。

	mysql> slave stop;

在主数据库插入。

	mysql> insert into sxit values('1','zhangsan');

在从数据库B插入。

	mysql> insert into sxit values('2','lisi');

在从数据库C插入。

	mysql> insert into sxit values('3','john');

登陆到amoeba服务器,进行读写分离的测试:

	[root@chenllcentos ~]# mysql -uroot -pyourpassword -h1XX.XX.XX.181 -P8066
	mysql> use test;
	mysql> select * from sxit;
	+------+------+
	| id   | name |
	+------+------+
	|    2 | lisi |
	+------+------+
	mysql> select * from sxit;
	+------+------+
	| id   | name |
	+------+------+
	|    3 | john |
	+------+------+

重复执行多次，发现始终只显示从数据库的数据，说明如果进行数据库读操作，Amoeba只将读数据SQL命令路由至从数据库。  
登录主数据库。

	[root@chenllcentos ~]# mysql -uroot -pyourpassword 
	mysql> use test;
	mysql> select * from sxit;
	+------+----------+
	| id   | name     |
	+------+----------+
	|    1 | zhangsan |
	+------+----------+

可以验证，使用Amoeba对Mysql读写分离成功。若此时开启从数据库主从复制，则可以进行Mysql集群和负载均衡。

## 小结

使用Amoeba做数据库代理，对于应用层来说是透明的。所谓透明，可以这么简单理解，是否使用代理，在应用层编码上是没有任何区别的，即使用代理的情况下，应用层和数据层能够保持高度解耦。