---
layout: post
title: "使用Nginx反向代理Tomcat并实现负载均衡"
date: 2014-11-18 15:45:17 +0800
comments: true
tags: Server
keywords: Nginx, Tomcat, Cluster, load balance
description: 使用Nginx反向代理Tomcat并实现负载均衡
---
## 引子
随着Web应用所面对的用户群不断壮大，很快就会出现Tomcat的性能瓶颈问题。  
此时面临着两种方案选择：  
1.升级主机硬件配置；  
2.搭建Tomcat集群。  
如果选择方案1，显然将面临高成本问题。面对类似问题，通常是采取方案2的做法。单机运行Tomcat所需要的服务器硬件配置要求并不高，更重要的是可以根据需求持续增加集群规模，以达到更高的性能要求。这种情况下，方案1是不可取的，也是无法办到的。  
既然需要搭建Tomcat集群，那么就需要专门的工具负责调度，Nginx就是这类工具的一种。Nginx是一款高性能的HTTP和反向代理服务器。字面上理解这句话有两层含义：  
1.Nginx是一个Web服务器，在这点上和Tomcat没有任何区别；  
2.Nginx还具有反向代理功能，Web服务器集群就是在此功能基础上实现的。<!-- more -->
## 反向代理
反向代理（Reverse Proxy）方式是指以代理服务器来接受Internet上的连接请求，然后将请求转发给内部网络上的服务器；并将从服务器上得到的结果返回给Internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。以上定义来自互联网。反向代理拓扑结构如图所示：

![](/images/2014/11/01.png)

反向代理服务器通常有两种模型，它可以作为内容服务器的替身，也可以作为内容服务器集群的负载均衡器。  
1.作为内容服务器替身  

![](/images/2014/11/02.png)

2.作为负载均衡器  

![](/images/2014/11/03.png)

在Tomcat集群中，用到的自然是Nginx作为负载均衡器。
以上只是对Nginx的简单介绍，若需要更多的Nginx信息，可以访问[Nginx官网](http://nginx.org/)。
## Nginx安装
一直牢记前辈的教诲：入门一项新技术，首先要做的是先用上它。那么，让我们先从安装Nginx开始。访问Nginx官网，进入[Nginx下载页](http://nginx.org/en/download.html)，选择最新的稳定版本。  
本文例子所用的安装环境：  

	[root@chenllcentos ~]# cat /etc/redhat-release 
	CentOS release 6.5 (Final)
	[root@chenllcentos ~]# uname -a
	Linux chenllcentos.localdomain 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

当然，安装Nginx之前，还需要安装好编译工具和第三方依赖包：

	[root@chenllcentos ~]# yum -y install gcc gcc-c++ autoconf automake
	[root@chenllcentos ~]# yum -y install zlib zlib-devel openssl openssl-devel pcre pcre-devel

接下来，开始进行Nginx源码安装。  
解压Nginx安装包：  

	[root@chenllcentos dev]# tar zxvf nginx-1.6.2.tar.gz

配置编译选项，这里直接使用默认值：

	[root@chenllcentos dev]# cd nginx-1.6.2
	[root@chenllcentos nginx-1.6.2]# ./configure

关于Nginx编译选项说明，可以查阅[Building nginx from Sources](http://nginx.org/en/docs/configure.html)。  
Nginx默认安装目录为`/usr/local/nginx`。  
编译并安装：

	[root@chenllcentos nginx-1.6.2]# make && make install

到此，Nginx的安装完成。  
启动Nginx：  

	[root@chenllcentos ~]# /usr/local/nginx/sbin/nginx

通过`ps`查看是否启动成功：

	[root@chenllcentos ~]# ps -ef | grep nginx
	root     31807     1  0 19:50 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
	root     31808 31807  0 19:50 ?        00:00:00 nginx: worker process      
	root     31809 31807  0 19:50 ?        00:00:00 nginx: worker process      
	root     31810 31807  0 19:50 ?        00:00:00 nginx: worker process      
	root     31811 31807  0 19:50 ?        00:00:00 nginx: worker process      
	root     31821  8578  0 19:56 pts/0    00:00:00 grep nginx

可以看到，Nginx有一个主线程`master process`和N个工作线程`worker process`。  
Nginx启动默认加载`/usr/local/nginx/conf/nginx.conf`配置文件，如果需要让Nginx加载我们指定的配置文件，可通过参数`-c`实现。例如：

	[root@chenllcentos ~]# /usr/local/nginx/sbin/nginx -c /data/mynginx.conf

同配置Tomcat服务器相同，如果要让远程主机可以访问Nginx，需要对Nginx监听的端口放行。那么，Nginx是监听哪个端口呢？让我们通过`netstat`看看：

	[root@chenllcentos ~]# netstat -ntlp | grep nginx
	tcp        0      0 0.0.0.0:80                  0.0.0.0:*                   LISTEN      31807/nginx  

Nginx默认的监听端口是80端口，可以通过`nginx.conf`配置文件的`listen`参数进行更改。

	[root@chenllcentos ~]# cat /usr/local/nginx/conf/nginx.conf | grep listen
	   listen       80;

修改`iptables`文件，添加如下内容：

	# 放行Nginx端口
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT

重启`iptables`服务：

	[root@chenllcentos ~]# service iptables restart

此时我们就能通过[http://youripaddress:80/]()访问Nginx了。如果配置正确，默认显示Nginx主页。

![](/images/2014/11/04.png)

关闭Nginx，一般情况是采用给Nginx主进程发送系统信号。例如：

	[root@chenllcentos ~]# kill -QUIT `cat /usr/local/nginx/nginx.pid`

其中`nginx.pid`是保存了Nginx主进程ID，这样就省去了查看Nginx进程号的步骤。更多的Nginx系统信号可以查阅[Controlling nginx](http://nginx.org/en/docs/control.html)。
## Nginx配置文件
在使用Nginx作为Web服务器或反向代理服务器时，Nginx的配置文件`nginx.conf`就是全部。Nginx的配置文件默认在Nginx安装目录的conf目录下，让我们看看它的主要组成部分。

	......
	events
	{
	......
	}
	 
	http
	{
	......
	    server
	    {
	    ......
	    }
	 
	    server
	    {
	    ......
	    }
	......
	}

不管Nginx实现的功能有多么复杂，配置文件都是由`events`、`http`和`server`这几个关键节点组成。这里先给出一个完整的配置实例，并对主要参数进行说明。

	# 使用的用户和组
	user  root root;
	# 指定工作衍生进程数（一般等于CPU的总核数或总核数的两倍，例如两个四核CPU，则总核数为8）
	worker_processes 8;
	# 指定错误日志存放的路径，错误日志记录级别可选项为：[ debug | info | notice | warn | error | crit ]
	error_log  /data1/logs/nginx_error.log  crit;
	# 指定 pid 存放的路径
	pid        /usr/local/nginx/nginx.pid;

	# 指定文件描述符数量
	worker_rlimit_nofile 51200;

	events
	{
	  # 使用的网络I/O模型，Linux系统推荐采用epoll模型，FreeBSD系统推荐采用kqueue模型
	  use epoll;
	 
	  # 允许的连接数
	  worker_connections 51200;
	}

	http
	{
	  include       mime.types;
	  default_type  application/octet-stream;
	  # 设置使用的字符集，如果一个网站有多种字符集，请不要随便设置，应让程序员在HTML代码中通过Meta标签设置
	  #charset  gb2312;
	      
	  server_names_hash_bucket_size 128;
	  client_header_buffer_size 32k;
	  large_client_header_buffers 4 32k;
	  
	  # 设置客户端能够上传的文件大小
	  client_max_body_size 8m;
	      
	  sendfile on;
	  tcp_nopush     on;

	  keepalive_timeout 60;

	  tcp_nodelay on;

	  fastcgi_connect_timeout 300;
	  fastcgi_send_timeout 300;
	  fastcgi_read_timeout 300;
	  fastcgi_buffer_size 64k;
	  fastcgi_buffers 4 64k;
	  fastcgi_busy_buffers_size 128k;
	  fastcgi_temp_file_write_size 128k;
	  # 开启gzip压缩
	  gzip on;
	  gzip_min_length  1k;
	  gzip_buffers     4 16k;
	  gzip_http_version 1.1;
	  gzip_comp_level 2;
	  gzip_types       text/plain application/x-javascript text/css application/xml;
	  gzip_vary on;

	  #limit_zone  crawler  $binary_remote_addr  10m;

	  server
	  {
	    listen       80;
	    server_name  www.yourdomain.com yourdomain.com;
	    index index.html index.htm index.php;
	    root  /data0/htdocs;

	    #limit_conn   crawler  20;
	    
	    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
	    {
	      expires      30d;
	    }

	    location ~ .*\.(js|css)?$
	    {
	      expires      1h;
	    }    

	    log_format  access  '$remote_addr - $remote_user [$time_local] "$request" '
	              '$status $body_bytes_sent "$http_referer" '
	              '"$http_user_agent" $http_x_forwarded_for';
	    access_log  /data1/logs/access.log  access;
	  }
	}

其中`location`元素可以通过正则表达式，可以对Web应用做动静态资源分离和设置缓存。例子中Nginx对图片视频类资源设置缓存30天，对js和css页面资源设置缓存1小时。
## 配置负载均衡
在nginx.conf配置文件中新增upstream节点，使用location拦截所有请求，规定负载均衡规后由Nginx转发给后台真实Web服务器即可实现负载均衡，达到Web服务器集群的效果。配置如下：

	user  root root;

	worker_processes 8;

	error_log  /data1/logs/nginx_error.log  crit;

	pid        /usr/local/nginx/nginx.pid;

	worker_rlimit_nofile 51200;

	events 
	{
	 use epoll;
	 worker_connections 51200;
	}

	http 
	{
	 include       mime.types;
	 default_type  application/octet-stream;

	 #charset  utf-8;
	     
	 server_names_hash_bucket_size 128;
	 client_header_buffer_size 32k;
	 large_client_header_buffers 4 32k;
	     
	 sendfile on;
	 #tcp_nopush     on;

	 keepalive_timeout 65;

	 tcp_nodelay on;

	 fastcgi_connect_timeout 300;
	 fastcgi_send_timeout 300;
	 fastcgi_read_timeout 300;
	 fastcgi_buffer_size 64k;
	 fastcgi_buffers 4 64k;
	 fastcgi_busy_buffers_size 128k;
	 fastcgi_temp_file_write_size 128k;

	 gzip on;
	 gzip_min_length  1k;
	 gzip_buffers     4 16k;
	 gzip_http_version 1.1;
	 gzip_comp_level 2;
	 gzip_types       text/plain application/x-javascript text/css application/xml;
	 gzip_vary on;

	 #limit_zone  crawler  $binary_remote_addr  10m;

	 #允许客户端请求的最大单个文件字节数
	 client_max_body_size     300m;

	 #缓冲区代理缓冲用户端请求的最大字节数，可以理解为先保存到本地再传给用户
	 client_body_buffer_size  128k;
	              
	 #跟后端服务器连接的超时时间_发起握手等候响应超时时间
	 proxy_connect_timeout    600;
	              		
	 #连接成功后_等候后端服务器响应时间_其实已经进入后端的排队之中等候处理
	 proxy_read_timeout       600;
	              
	 #后端服务器数据回传时间_就是在规定时间内后端服务器必须传完所有的数据
	 proxy_send_timeout       600;
	              
	 #代理请求缓存区_这个缓存区间会保存用户的头信息以供Nginx进行规则处理_一般只要能保存下头信息即可
	 proxy_buffer_size        16k;
	              
	 #同上 告诉Nginx保存单个用的几个Buffer 最大用多大空间
	 proxy_buffers            4 32k;
	              
	 #如果系统很忙的时候可以申请更大的proxy_buffers 官方推荐*2	
	 proxy_busy_buffers_size 64k;
	              
	 #proxy缓存临时文件的大小
	 proxy_temp_file_write_size 64k;
	 
	 upstream tomcat_server_pool {
	   server   1XX.XX.XX.182:8080 weight=1 max_fails=2 fail_timeout=30s;
	   server   1XX.XX.XX.182:8081 weight=1 max_fails=2 fail_timeout=30s;
	   server   1XX.XX.XX.183:8080 weight=1 max_fails=2 fail_timeout=30s;
	 }

	 #第一个虚拟主机，反向代理tomcat_server_pool这组服务器
	 server
	 {
	   listen       80;
	   server_name  1XX.XX.XX.181;

	   location /
	   {
	         #如果后端的服务器返回502、504、执行超时等错误，自动将请求转发到upstream负载均衡池中的另一台服务器，实现故障转移。
	         proxy_next_upstream http_502 http_504 error timeout invalid_header;
	         proxy_pass http://tomcat_server_pool;
	         proxy_set_header Host  1XX.XX.XX.181;
	         proxy_set_header X-Forwarded-For  $remote_addr;
	   }

	   access_log  /data1/logs/1XX.XX.XX.181_access.log;
	 }
	}

通过上述配置文件可以看出，Nginx运行在1XX.XX.XX.181主机并监听80端口，负责拦截所有请求并以轮询的方式将请求转发给后台3台Tomcat服务器。其中1XX.XX.XX.182主机运行2个Tomcat实例，分别监听8080端口和8081端口；1XX.XX.XX.183主机运行1个Tomcat实例，监听8080端口。由于Nginx负责将请求转发给Tomcat服务器，因此必须开放相应的监听端口。  
开放1XX.XX.XX.182主机8080和8081端口：

	[root@chenllcentos ~]# vi /etc/sysconfig/iptables

添加如下内容：

	# 放行Tomcat端口
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT
	-A INPUT -m state --state NEW -m tcp -p tcp --dport 8081 -j ACCEPT

重启`iptables`服务：

	[root@chenllcentos ~]# service iptables restart

同样地，开放1XX.XX.XX.183主机8080端口。现在重新访问Nginx主页[http://youripaddress:80/]()：

![](/images/2014/11/05.png)

出现Tomcat默认主页，说明Nginx反向代理成功。