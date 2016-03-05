---
layout: post
title: "Tomcat集群性能测试与分析"
date: 2014-11-19 11:00:47 +0800
comments: true
tags: Server
keywords: AB, Tomcat, Cluster, Benckmark
description: Tomcat集群性能测试与分析
---
## 引子
在[使用Nginx反向代理Tomcat并实现负载均衡](http://aaronchansunny.github.io/blog/2014/11/18/nginx-tomcat-cluster/)博客中，我们使用Nginx搭建Tomcat简单集群，并实现负载均衡。之所有进行Web服务器集群，就是希望能够达到水平扩展的目的。当集群中新增新的Web服务器，理论上整个Web应用的并发量就能得到线性增长。现在以简单的注册为例，来验证我们之前的集群方案是否有效。<!-- more -->
## 测试环境
1.1XX.XX.XX.181  
运行Nginx、Tomcat和MySQL  
2.1XX.XX.XX.182   
运行Tomcat  
1XX.XX.XX.183  
运行Tomcat  
服务器环境：

	[root@chenllcentos ~]# cat /etc/redhat-release 
	CentOS release 6.5 (Final)
	[root@chenllcentos ~]# uname -a
	Linux chenllcentos.localdomain 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

## 部署应用
将Web项目文件分别拷贝至服务器1XX.XX.XX.181、1XX.XX.XX.182和1XX.XX.XX.183，这里放在`/data0/htdocs/www`目录下。  
接下来，设置Tomcat配置文件的虚拟目录如下：

	[root@chenllcentos ~]# cat /usr/local/tomcat/conf/server.xml | grep appBase
	      <Host name="localhost"  appBase="/data0/htdocs/www/"

## Tomcat集群测试
现在通过模拟1000用户并发执行注册操作，持续时间30秒来测试服务器集群性能，同时打印系统资源占用信息。测试AB命令如下；

	[root@chenllcentos ~]# ab -c1000 -t30 -p /data/post.txt http://1XX.XX.XX.181/chatMeServer/chatMeAction_registration.rjaction

### 1台Tomcat性能
首先只启动1XX.XX.XX.182主机Tomcat一台Web服务器，测试数据如下：

	Concurrency Level:      1000
	Time taken for tests:   30.000 seconds
	Complete requests:      14843
	Failed requests:        0
	Write errors:           0
	Total transferred:      5640340 bytes
	Total POSTed:           3564675
	HTML transferred:       1513986 bytes
	Requests per second:    494.76 [#/sec] (mean)
	Time per request:       2021.163 [ms] (mean)
	Time per request:       2.021 [ms] (mean, across all concurrent requests)
	Transfer rate:          183.60 [Kbytes/sec] received
	                        116.04 kb/s sent
	                        299.64 kb/s total

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:        0    6  20.5      0     102
	Processing:   103 1947 353.9   2008    3379
	Waiting:      102 1947 353.9   2007    3379
	Total:        204 1953 348.6   2008    3455

主机1XX.XX.XX.181系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all      5.82      0.00     28.84      0.00      0.26     65.08

主机1XX.XX.XX.182系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     37.07      0.00     24.80      0.00      0.27     37.87

### 2台Tomcat性能
接下来启动1XX.XX.XX.183主机Tomcat服务器，现在共运行2台Web服务器，测试数据如下；

	Concurrency Level:      1000
	Time taken for tests:   30.000 seconds
	Complete requests:      25312
	Failed requests:        0
	Write errors:           0
	Keep-Alive requests:    0
	Total transferred:      9618560 bytes
	Total POSTed:           6551439
	HTML transferred:       2581824 bytes
	Requests per second:    843.73 [#/sec] (mean)
	Time per request:       1185.220 [ms] (mean)
	Time per request:       1.185 [ms] (mean, across all concurrent requests)
	Transfer rate:          313.10 [Kbytes/sec] received
	                        213.26 kb/s sent
	                        526.36 kb/s total

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:        0    3  15.4      0     113
	Processing:     5 1141 1139.4    650    3879
	Waiting:        5 1140 1139.4    650    3878
	Total:          6 1144 1140.2    703    3916

主机1XX.XX.XX.181系统（运行Nginx和Mysql）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     10.05      0.00     47.09      0.00      0.26     42.59

主机1XX.XX.XX.182（运行Tomcat）系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     36.22      0.00     23.24      0.00      0.27     40.27

主机1XX.XX.XX.183系统（运行Tomcat）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     52.86      0.00     22.92      0.00      0.26     23.96

###3台Tomcat性能
如果主机1XX.XX.XX.183再开一个Tomcat实例呢？测试数据：

	Concurrency Level:      1000
	Time taken for tests:   30.000 seconds
	Complete requests:      42718
	Failed requests:        0
	Write errors:           0
	Keep-Alive requests:    0
	Total transferred:      16243430 bytes
	Total POSTed:           10885284
	HTML transferred:       4357236 bytes
	Requests per second:    1423.93 [#/sec] (mean)
	Time per request:       702.283 [ms] (mean)
	Time per request:       0.702 [ms] (mean, across all concurrent requests)
	Transfer rate:          528.76 [Kbytes/sec] received
	                        354.34 kb/s sent
	                        883.09 kb/s total

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:        0    2  11.6      0     109
	Processing:     3  685 831.2    431    7105
	Waiting:        2  685 831.2    430    7105
	Total:          3  687 834.8    434    7201

主机1XX.XX.XX.181系统（运行Nginx和Mysql）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     10.05      0.00     47.09      0.00      0.26     42.59

主机1XX.XX.XX.182（运行Tomcat）系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     36.22      0.00     23.24      0.00      0.27     40.27

主机1XX.XX.XX.183系统（运行Tomcat和Tomcat1）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     66.00      0.00     33.50      0.00      0.00      0.50 --->CPU达到瓶颈

可以发现，此时主机1XX.XX.XX.183的CPU达到瓶颈。从AB测试结果`Complete requests`和`Time per request`看，虽然多运行了一个Tomcat实例，但由于1XX.XX.XX.183主机CPU已经达到瓶颈，整个集群的性能并没有成线性增长。  
如果将第三台Tomcat搭建在1XX.XX.XX.181主机呢？测试数据：

	Concurrency Level:      1000
	Time taken for tests:   30.018 seconds
	Complete requests:      33026
	Failed requests:        0
	Write errors:           0
	Keep-Alive requests:    0
	Total transferred:      12622460 bytes
	Total POSTed:           8389557
	HTML transferred:       3388134 bytes
	Requests per second:    1100.21 [#/sec] (mean)
	Time per request:       908.914 [ms] (mean)
	Time per request:       0.909 [ms] (mean, across all concurrent requests)
	Transfer rate:          410.64 [Kbytes/sec] received
	                        272.94 kb/s sent
	                        683.58 kb/s total

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:        0  151 159.1    116    1359
	Processing:     3  746 887.7    452    7192
	Waiting:        3  694 887.5    385    7183
	Total:          3  896 888.1    665    7257

主机1XX.XX.XX.181系统（运行Nginx、Tomcat和Mysql）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     56.64      0.00     43.11      0.00      0.25      0.00 --->CPU达到瓶颈

主机1XX.XX.XX.182（运行Tomcat）系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     29.54      0.00     15.18      0.00      0.27     55.01

主机1XX.XX.XX.183系统（运行Tomcat）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     25.74      0.00     17.69      0.00      0.27     56.30

通过`sar`数据可以知道，此时主机1XX.XX.XX.181的CPU已达到瓶颈，性能也没有成比例增长。  
### 测试小结
通过上述测试结果可以看出，只要物理主机资源还未达到瓶颈，`Complete requests`和`Time per request`都能得到近乎线性的增长，因此可以证明我们的集群方案是有效的。
## 使用Tomcat集群提高并发数
使用AB测试工具模拟1万用户同时并发注册的情况，持续时间30秒：

	[root@chenllcentos ~]# ab -c10000 -t30 -k -p /data/post.txt http://1XX.XX.XX.181/chatMeServer/chatMeAction_registration.rjaction

### 1台Tomcat性能
1台Tomcat的测试数据：

	Concurrency Level:      10000
	Time taken for tests:   30.000 seconds
	Complete requests:      13791
	Failed requests:        11 --->出现请求响应失败
	   (Connect: 0, Receive: 0, Length: 11, Exceptions: 0)
	Write errors:           0
	Non-2xx responses:      11
	Keep-Alive requests:    11
	Total transferred:      5240008 bytes
	Total POSTed:           5923710
	HTML transferred:       1407452 bytes
	Requests per second:    459.70 [#/sec] (mean)
	Time per request:       21753.394 [ms] (mean)
	Time per request:       2.175 [ms] (mean, across all concurrent requests)
	Transfer rate:          170.57 [Kbytes/sec] received
	                        192.83 kb/s sent
	                        363.40 kb/s total

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:        0  408 340.2    388    1030
	Processing:     1 13053 5949.5  14975   26485
	Waiting:        1 13052 5949.7  14975   26485
	Total:          2 13460 5715.0  15317   26765

主机1XX.XX.XX.181系统（运行Nginx和Mysql）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all      7.20      0.00     27.20      0.00      0.00     65.60

主机1XX.XX.XX.182（运行Tomcat）系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     36.31      0.00     25.75      0.00      0.00     37.94

可以看出，此时Tomcat出现11次请求响应失败。我们往集群新增一台Tomcat，看是否能解决响应失败问题。  
### 2台Tomcat性能
2台Tomcat的测试数据：

	Concurrency Level:      10000
	Time taken for tests:   30.002 seconds
	Complete requests:      24053 --->给定时间处理的请求量成比例增长，说明负载均衡有效
	Failed requests:        0 --->解决了请求失败问题
	Write errors:           0
	Keep-Alive requests:    0
	Total transferred:      9143180 bytes
	Total POSTed:           8478201
	HTML transferred:       2454222 bytes
	Requests per second:    801.71 [#/sec] (mean)
	Time per request:       12473.334 [ms] (mean)
	Time per request:       1.247 [ms] (mean, across all concurrent requests)
	Transfer rate:          297.61 [Kbytes/sec] received
	                        275.96 kb/s sent
	                        573.57 kb/s total

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:        0  244 317.4      8    1004
	Processing:   274 9617 2913.4  10995   16630
	Waiting:      268 9616 2913.8  10994   16629
	Total:        688 9861 2674.2  11020   16931

主机1XX.XX.XX.181系统（运行Nginx和Mysql）资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     20.82      0.00     49.36      0.51      0.51     28.79

主机1XX.XX.XX.182（运行Tomcat）系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     36.96      0.00     22.55      0.00      0.00     40.49

主机1XX.XX.XX.183（运行Tomcat）系统资源占用：

	[root@chenllcentos ~]# sar -u 1
	CPU     %user     %nice   %system   %iowait    %steal     %idle
	all     55.73      0.00     21.88      0.00      0.00     22.40

此时不仅`Complete requests`成比例增长，同时也解决了服务器响应请求失败的问题，说明Tomcat集群有效。