---
layout: post
title: "Eclipse使用Maven搭建Spring+Struts2+Hibernate工程"
date: 2014-11-21 14:46:21 +0800
comments: true
tags: JavaWeb
keywords: Maven, Eclipse, Spring, Struts2, Hibernate
description: Eclipse使用Maven搭建Spring+Struts2+Hibernate工程
---
## 引子
为了熟悉Spring+Struts2+Hibernate工程（SSH2）各个框架的配置，一直想自己手动搭个SSH2。但每次一想到那么多jar包依赖，总提不起勇气，相信这也是很多刚学习JavaWeb的朋友所困扰的问题。对于Maven真是久仰大名，最近拜读了许晓斌前辈《Maven实战》这本书，真是有种相见恨晚的感觉。总之，Maven为我们建立了一套标准，并提供中央仓库让我们能在线获取第三方jar包资源，只要我们遵循这个标准，构建将变得如此轻松。使用Maven构建的最佳实践是使用命令行，但对于Java开发人员，Eclipse等IDE工具虽然在构建上具有先天不足，但在编码上确实能大大提高效率。幸运的是，Eclipse现在已经集成m2eclipse插件，提供Maven工程创建和管理。那就现学现卖吧~就让我们一起在Eclipse下用Maven搭建一个SSH2工程示例。<!-- more -->
## 安装Maven
安装Maven很简单，首先访问Maven官网，选择最新版本下载，安装过程一路`next`即可。  
安装结束后，添加Maven安装目录到系统路径PATH。  
运行`sysdm.cpl`打开系统属性：

![](/images/2014/11/06.png)

选择**高级**选项卡：  

![](/images/2014/11/07.png)

在`PATH`中添加Maven安装目录，例如：`E:\Program Files\maven-3.2.3\bin`。
验证Maven是否安装成功：  

	C:\Users\OA>mvn --version
	Apache Maven 3.2.3 (33f8c3e1027c3ddde99d3cdebad2656a31e8fdf4; 2014-08-12T04:58:1
	0+08:00)
	Maven home: E:\Program Files\maven-3.2.3\bin\..
	Java version: 1.8.0_05, vendor: Oracle Corporation
	Java home: E:\Program Files\Java\jdk1.8.0_05\jre
	Default locale: zh_CN, platform encoding: GBK
	OS name: "windows 7", version: "6.1", arch: "x86", family: "dos"

## 创建Maven工程
### Eclipse版本
如果Eclipse版本过低，可能还没有集成`m2eclipse`插件，需要手动安装。例子中使用的Eclipse版本：

![](/images/2014/11/08.png)

### 具体步骤
Luna版Eclipse默认已经集成插件，可以直接创建Maven工程：

![](/images/2014/11/09.png)

![](/images/2014/11/10.png)

直接Next，在Filter中输入web关键字，找到maven-archetype-webapp：

![](/images/2014/11/11.png)

继续Next，出现如下界面：

![](/images/2014/11/12.png)

依次输入Group Id和Artifact Id信息：

![](/images/2014/11/13.png)

Package会自动补全，点击Finish。
### 工程目录结构
Eclipse创建好的Maven工程目录结构如下：

![](/images/2014/11/14.png)

额，怎么还有错误，而且目录结构和Maven约定的也不相同。没关系，让我们一步步解决他们。   
使用Maven自动构建和管理项目有一个前提，目录结构需要和Maven规定的相一致。  
通过Eclipse生成的Maven工程还差以下目录：  
1.src/main/java  
2.src/test/java  
3.src/test/resources  
这三个目录，需要我们手动添加，添加后Java Resources目录如下：

![](/images/2014/11/15.png)

右键工程Properties->Java Build Path在Source选项卡配置Source folders输出目录：

![](/images/2014/11/16.png)

现在我们的工程目录就和Maven标准目录相一致了。
### 错误解决
接下来， 让我们看看工程报的错误：

![](/images/2014/11/17.png)

提示说找不到`javax.servlet.http.HttpServlet`，也就说缺少Servlet API。按照传统的做法，我们会新建一个lib目录，然后网上找到Servlet API的jar包，还要配置工程属性，将依赖添加进来。而且，每次新增第三方库，都要重复上述步骤，这既无趣又耗时。在依赖库版本更新时，还要同时更新相应的依赖包，这时候特别容易出现版本冲突问题。但是如果有了Maven呢？一切变得如此简单，我们只需在POM中添加如下依赖信息：

	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>servlet-api</artifactId>
		<version>2.5</version>
		<scope>provided</scope>
	</dependency>

接下来，把所有事情交给Maven吧。保存POM文件后，Maven会根据依赖信息，先到本地仓库加载依赖包，如果本地没有所需依赖包，再到Maven中央仓库下载。依赖包下载完成后，刚才的错误也就解决了。  
Luna创建Maven工程默认使用JDK1.5，因此出现以下警告信息：

![](/images/2014/11/18.png)

右键工程Properties->Java Compiler将Compiler compliance level更改为1.8：

![](/images/2014/11/19.png)

> 这里需要根据实际安装的JDK版本进行修改，例子中安装的是JDK1.8。

保存退出后，如下错误信息：

![](/images/2014/11/20.png)

右键工程Properties->Project Facets，将Java版本更改为1.8：

![](/images/2014/11/21.png)

至此，使用Maven搭建JavaWeb工程全部完成。
## 配置POM文件
使用Maven管理SSH2所需要的依赖库，POM文件就是全部，具体可以查看[pom.xml](https://github.com/aaronchansunny/mavendemo/blob/master/pom.xml)。修改POM文件后保存，m2eclipse插件就开始从Maven中央库下载依赖包，这个过程取决于你的网速，如果依赖包较多，需要的时间比较漫长。如果Maven本地仓库已经有这些依赖包了，Maven将直接加载本地包。那么，要如何找到我们需要的第三方库呢？一般的做法是直接谷歌，例如我们要找spring框架依赖，直接输入关键字**maven repository spring**：

![](/images/2014/11/22.png)

也可以直接访问：[http://mvnrepository.com/](http://mvnrepository.com/)进行搜索。
## SSH2配置文件
到目前为止，Maven已经完成它的依赖包管理使命了，以后要是想新增或更新依赖包直接修改POM文件即可，相对传统管理依赖包的方式简直是一本万利。接下来，进行SSH2各模块的配置，示例程序中已经给出详细注释，这里只简单列出说明。  

- web.xml

```
	<?xml version="1.0" encoding="UTF-8"?>
	<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
		xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
		id="WebApp_ID" version="3.0">
		<display-name>ssh2_test</display-name>
		<welcome-file-list>
			<welcome-file>index.jsp</welcome-file>
		</welcome-file-list>

		<filter>
			<filter-name>CharsetEncodingFilter</filter-name>
			<filter-class>com.fish.ssh2.filter.CharsetEncodingFilter</filter-class>
			<init-param>
				<param-name>encoding</param-name>
				<param-value>GB18030</param-value>
			</init-param>
		</filter>

		<filter-mapping>
			<filter-name>CharsetEncodingFilter</filter-name>
			<url-pattern>/*</url-pattern>
		</filter-mapping>
		<!-- 配置spring资源 -->
		<context-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:config/applicationContext-*.xml</param-value>
		</context-param>

		<!-- 配置CharacterEncoding，设置字符集 -->
		<!-- <filter> <filter-name>characterEncodingFilter</filter-name> <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class> 
			<init-param> <param-name>encoding</param-name> <param-value>UTF-8</param-value> 
			</init-param> <init-param> <param-name>forceEncoding</param-name> <param-value>true</param-value> 
			</init-param> </filter> <filter-mapping> <filter-name>characterEncodingFilter</filter-name> 
			<url-pattern>/*</url-pattern> </filter-mapping> -->




		<!-- 将HibernateSession开关控制配置在Filter，保证一个请求一个session，并对lazy提供支持 -->
		<filter>
			<filter-name>OpenSessionInView</filter-name>
			<filter-class>org.springframework.orm.hibernate4.support.OpenSessionInViewFilter</filter-class>
			<init-param>
				<param-name>singleSession</param-name>
				<param-value>true</param-value>
			</init-param>

		</filter>

		<filter-mapping>
			<filter-name>OpenSessionInView</filter-name>
			<url-pattern>/*</url-pattern>
		</filter-mapping>

		<!-- 配置Struts2 -->
		<filter>
			<filter-name>struts2</filter-name>
			<filter-class>org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter</filter-class>
			<init-param>
				<param-name>config</param-name>
				<param-value>struts-default.xml,struts-plugin.xml,/config/struts.xml</param-value>
			</init-param>
		</filter>

		<filter-mapping>
			<filter-name>struts2</filter-name>
			<url-pattern>/*</url-pattern>
		</filter-mapping>


		<!-- 配置spring -->
		<listener>
			<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
		</listener>

	</web-app>
```

web.xml文件是Web容器加载Web应用的入口，主要包括应用全局变量、Spring监听器、Struts2核心控制器以及设置字符集等信息。  

- applicationContext-beans.xml

```
	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
		xmlns:context="http://www.springframework.org/schema/context" xmlns:tx="http://www.springframework.org/schema/tx"
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
			http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.2.xsd
			http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd
			http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.2.xsd">

		<!-- Spring管理Struts2的Action -->
		<bean name="loginAction" class="com.fish.ssh2.action.LoginAction" scope="prototype"></bean>
		<bean name="userMainAction" class="com.opensymphony.xwork2.ActionSupport" scope="prototype"></bean>
		<bean name="userAction" class="com.fish.ssh2.action.UserAction" scope="prototype">
			<!-- <property name="userManage" ref="userManage"></property> -->
		</bean>

		<!-- Spring管理Struts2的Interceptor -->
		<bean name="checkLoginInterceptor" class="com.fish.ssh2.interceptor.CheckLogin" scope="prototype"></bean>

		
		
		<bean name="userManage" class="com.fish.ssh2.service.impl.UserManageImpl">
			<!-- <property name="userDao" ref="userDao"></property> -->
		</bean>
		
		<bean name="userDao" class="com.fish.ssh2.dao.impl.UserDaoImpl">
			<property name="sessionFactory" ref="sessionFactory" />
		</bean>
		
	</beans>
```

Spring在SSH2框架中就是最大的工厂，所有交给Spring托管的Bean和它们之间的依赖关系都要在applicationContext-beans.xml声明。  

- applicationContext-common.xml

```
	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
		xmlns:context="http://www.springframework.org/schema/context" xmlns:tx="http://www.springframework.org/schema/tx"
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
			http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.2.xsd
			http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd
			http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.2.xsd">

		<!-- 启用spring注解支持 -->
		<context:annotation-config />

		<!--配数据源 -->
		<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
			destroy-method="close">
			<!-- 数据库连接驱动 -->
			<property name="driverClassName" value="com.mysql.jdbc.Driver" />
			<!-- 数据库url -->
			<property name="url"
				value="jdbc:mysql://localhost/db?characterEncoding=utf-8&amp;autoReconnect=true" />
			<!-- 数据库用户名 -->
			<property name="username" value="root" />
			<!-- 数据库密码 -->
			<property name="password" value="root" />
		</bean>

		<bean id="sessionFactory"
			class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
			<property name="dataSource" ref="dataSource" />

			<property name="hibernateProperties">
				<props>
					<!-- 指定数据库方言 -->
					<prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
					<!-- 是否显示Hibernate产生的SQL语句 -->
					<prop key="hibernate.show_sql">true</prop>
					<!-- 启动应用时，是否根据HBM文件创建数据表 -->
					<prop key="hibernate.hbm2ddl.auto">update</prop>
				</props>
			</property>
			<!-- 如果使用配置文件 -->
			<!-- <property name="mappingLocations"> <list> <value>classpath:com/jialin/entity/User.hbm.xml</value> 
				</list> </property> -->
			<property name="annotatedClasses">
				<list>
					<value>com.fish.ssh2.entity.User</value>
				</list>
			</property>

		</bean>

		<!-- 配置事务管理器 -->
		<bean id="transactionManager"
			class="org.springframework.orm.hibernate4.HibernateTransactionManager">
			<property name="sessionFactory" ref="sessionFactory" />
		</bean>

		<!-- 事务的传播特性 -->
		<tx:advice id="txadvice" transaction-manager="transactionManager">
			<tx:attributes>
				<tx:method name="*" propagation="REQUIRED" />
				<!--hibernate4必须配置为开启事务 否则 getCurrentSession()获取不到 -->
				<!-- <tx:method name="*" propagation="REQUIRED" read-only="true" /> -->
			</tx:attributes>
		</tx:advice>

		<!-- 那些类那些方法使用事务 -->
		<aop:config>
			<!-- 只对业务逻辑层实施事务 -->
			<aop:pointcut id="allManagerMethod"
				expression="execution(* com.fish.ssh2.service.impl.*.*(..))" />
			<aop:advisor pointcut-ref="allManagerMethod" advice-ref="txadvice" />
		</aop:config>

	</beans>
```

Spring的基本配置文件，包括配置常量定义、数据源和事务管理等。  

- struts.xml

```
	<?xml version="1.0" encoding="UTF-8" ?>
	<!DOCTYPE struts PUBLIC
	    "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN"
	    "http://struts.apache.org/dtds/struts-2.0.dtd">

	<struts>
		
		<!-- 将Action的创建交给spring来管理 -->  
	    <constant name="struts.objectFactory" value="spring" />  
		
		<!-- 更改struts2请求Action的后缀名，默认为action。若想去掉后缀，设为","即可 -->
		<constant name="struts.action.extension" value=","></constant>

		<!-- 公共包 -->
	<!-- 	<package name="abstract_struts" abstract="true" extends="struts-default"
			namespace="/">
			<interceptors>
				<interceptor name="checkLogin" class="com.fish.ssh2.interceptor.CheckLogin" />
				<interceptor-stack name="myInterceptor">
					<interceptor-ref name="checkLogin" />
					<interceptor-ref name="defaultStack" />
				</interceptor-stack>
			</interceptors>

			<default-interceptor-ref name="myInterceptor" />

			<global-results>
				<result name="checkLoginFail">/login.jsp</result>
			</global-results>
		</package> -->

		<package name="abstract_struts" abstract="true" extends="struts-default"
			namespace="/">
			<interceptors>
				<interceptor name="checkLogin" class="checkLoginInterceptor" />
				<interceptor-stack name="myInterceptor">
					<interceptor-ref name="checkLogin" />
					<interceptor-ref name="defaultStack" />
				</interceptor-stack>
			</interceptors>

			<!-- <default-interceptor-ref name="myInterceptor" /> -->

			<global-results>
				<result name="checkLoginFail">/login.jsp</result>
			</global-results>
		</package>
		

		<!-- 包含的配置文件 -->
		<include file="/config/struts-user.xml"></include>

	</struts>
```

Struts2核心配置文件。  

- struts-user.xml

```
	<?xml version="1.0" encoding="UTF-8" ?>
	<!DOCTYPE struts PUBLIC
	    "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN"
	    "http://struts.apache.org/dtds/struts-2.0.dtd">

	<struts>
	    <!-- 未与spring集成的写法 -->
		<!-- <package name="loginAction" namespace="/" extends="abstract_struts">
		
			<action name="login" class="com.fish.ssh2.action.LoginAction">
				<result name="success" type="redirect">userMain</result>
				<result name="fail">/fail.jsp</result>
			</action>

			该action只负责跳转，用struts提供的ActionSupport
			<action name="userMain" class="com.opensymphony.xwork2.ActionSupport">
				<result name="success">/userMain.jsp</result>
				<interceptor-ref name="myInterceptor" />
			</action>
		</package>

		<package name="userActions" namespace="/user" extends="abstract_struts">
			<action name="*_*" class="com.fish.ssh2.action.UserAction" method="{1}">
				<result name="success" type="redirect">/{2}.jsp</result>
				<result name="fail">/fail.jsp</result>
				<interceptor-ref name="myInterceptor" />
			</action>
		</package> -->
		
		<!-- 与spring集成的写法，action等交予spring管理 -->
		<package name="loginAction" namespace="/" extends="abstract_struts">
		
			<action name="login" class="loginAction">
				<result name="success" type="redirect">userMain</result>
				<result name="fail">/fail.jsp</result>
			</action>

			<!-- 该action只负责跳转，用struts提供的ActionSupport -->
			<action name="userMain" class="userMainAction">
				<result name="success">/userMain.jsp</result>
				<interceptor-ref name="myInterceptor" />
			</action>
		</package>

		<package name="userActions" namespace="/user" extends="abstract_struts">
			<action name="*_*" class="userAction" method="{1}">
				<result name="success" type="redirect">/{2}.jsp</result>
				<result name="fail">/fail.jsp</result>
				<interceptor-ref name="myInterceptor" />
			</action>
		</package>

</struts>
```

该配置文件用来定义业务逻辑Actions。  
可以发现，这里我们对Spring和Struts2的配置文件都进行拆分处理，这样做可以避免单个配置文件过于臃肿，也方便将来进行扩展。
## 创建数据库
在之前的数据源配置中，我们连接了一个名叫db的数据库：

	<!-- 数据库url -->
	<property name="url"
		value="jdbc:mysql://localhost/db?characterEncoding=utf-8&amp;autoReconnect=true" />
	<!-- 数据库用户名 -->
	<property name="username" value="root" />
	<!-- 数据库密码 -->
	<property name="password" value="root" />

因此，这里需要创建一个db数据库，并添加p_user表作为新增用户测试。  
登录MySQL数据库：

	C:\Users\OA> mysql -uroot -proot

执行如下SQL脚本：

	CREATE DATABASE db;
	USE db;
	DROP TABLE IF EXISTS `p_user`;
	CREATE TABLE `p_user` (
	  `id` int(10) unsigned AUTO_INCREMENT NOT NULL,
	  `name` varchar(45) DEFAULT NULL,
	  `password` varchar(45) DEFAULT NULL,
	  `age` int(11) DEFAULT 0,
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;

## 示例程序编写
为了简单起见，这里以一个简单的登录和新增用户作为例子进行说明。MVC分层结构如下：

![](/images/2014/11/23.png)

示例源码托管在Github：[https://github.com/aaronchansunny/mavendemo](https://github.com/aaronchansunny/mavendemo)
## 示例演示
### 登录  

![](/images/2014/11/24.png)

输入用户名：root；密码：root123。  
### 新增用户  

![](/images/2014/11/25.png)

新增新用户：test。  
### 查看数据库  

	mysql> use db;
	mysql> select * from p_user;
	+----+------+----------+------+
	| id | name | password | age  |
	+----+------+----------+------+
	|  1 | test | 123      |   12 |
	+----+------+----------+------+
	1 row in set (0.00 sec)

刚才新增的用户，已经成功保存至数据库。