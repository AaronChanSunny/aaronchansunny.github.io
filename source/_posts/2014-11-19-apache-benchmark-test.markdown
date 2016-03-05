---
layout: post
title: "使用Apache Benchmark测试Web服务器性能"
date: 2014-11-19 10:14:38 +0800
comments: true
tags: Server
keywords: Apache Benchmark, test, Web Server, ab
description: 使用Apache Benchmark测试Web服务器性能
---
## 引子
当Web应用正式上线之前，对Web服务器性能的预估是必不可少的。为了让运维人员对服务器的性能瓶颈有基本预估，需要对服务器进行压力测试。  
工欲善其事，必先利其器。Apache Benchmark（以下简称AB）作为Apache旗下的Http Server性能评测工具，使用方法非常简单，并且在高并发的情况下并不会占用太多系统资源。此外，AB可以直接在Web服务器本地发起测试请求。这点至关重要，因为我们希望测试的是服务器处理时间，而不包含数据的网络传输时间以及用户PC本地的计算时间。<!-- more -->
## 测试环境

	[root@chenllcentos ~]# cat /etc/redhat-release 
	CentOS release 6.5 (Final)
	[root@chenllcentos ~]# uname -a
	Linux chenllcentos.localdomain 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

## 安装AB
在CentOS下可以通过`yum`安装AB：

	yum -y install httpd-tools

## 基本使用
AB安装完毕，最简单的使用方法是：

	[root@chenllcentos ~]# ab http://www.baidu.com/
	[root@chenllcentos ~]# ab http://www.baidu.com/
	This is ApacheBench, Version 2.3 <$Revision: 655654 $>
	Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
	Licensed to The Apache Software Foundation, http://www.apache.org/

	Benchmarking www.baidu.com (be patient).....done


	Server Software:        BWS/1.1
	Server Hostname:        www.baidu.com
	Server Port:            80

	Document Path:          /
	Document Length:        82654 bytes

	Concurrency Level:      1
	Time taken for tests:   0.242 seconds
	Complete requests:      1
	Failed requests:        0
	Write errors:           0
	Total transferred:      83528 bytes
	HTML transferred:       82654 bytes
	Requests per second:    4.14 [#/sec] (mean)
	Time per request:       241.761 [ms] (mean)
	Time per request:       241.761 [ms] (mean, across all concurrent requests)
	Transfer rate:          337.40 [Kbytes/sec] received

	Connection Times (ms)
	              min  mean[+/-sd] median   max
	Connect:       15   15   0.0     15      15
	Processing:   215  215   0.0    215     215
	Waiting:       25   25   0.0     25      25
	Total:        230  230   0.0    230     230

上述测试结果参数说明：

	Server Software:        web服务器软件及版本
	Server Hostname:        请求的地址
	Server Port:            请求的端口

	Document Path:          请求的页面路径
	Document Length:        页面大小

	Concurrency Level:      并发数
	Time taken for tests:   测试总共花费的时间
	Complete requests:      完成的请求数
	Failed requests:        失败的请求数
	Write errors:           写入错误
	Total transferred:      总共传输字节数，包含http的头信息等
	HTML transferred:       html字节数，实际的页面传递字节数
	Requests per second:    每秒处理的请求数，服务器的吞吐量（重要）
	Time per request:       平均数，用户平均请求等待时间
	Time per request:       服务器平均处理时间
	Transfer rate:          平均传输速率（每秒收到的速率）

## AB运行参数
接下来，让我们看看`ab`详细使用参数，直接输入`ab`：

	[root@chenllcentos ~]# ab
	ab: wrong number of arguments
	Usage: ab [options] [http[s]://]hostname[:port]/path
	Options are:
	    -n requests     Number of requests to perform
	    -c concurrency  Number of multiple requests to make
	    -t timelimit    Seconds to max. wait for responses
	    -b windowsize   Size of TCP send/receive buffer, in bytes
	    -p postfile     File containing data to POST. Remember also to set -T
	    -u putfile      File containing data to PUT. Remember also to set -T
	    -T content-type Content-type header for POSTing, eg.
	                    'application/x-www-form-urlencoded'
	                    Default is 'text/plain'
	    -v verbosity    How much troubleshooting info to print
	    -w              Print out results in HTML tables
	    -i              Use HEAD instead of GET
	    -x attributes   String to insert as table attributes
	    -y attributes   String to insert as tr attributes
	    -z attributes   String to insert as td or th attributes
	    -C attribute    Add cookie, eg. 'Apache=1234. (repeatable)
	    -H attribute    Add Arbitrary header line, eg. 'Accept-Encoding: gzip'
	                    Inserted after all normal header lines. (repeatable)
	    -A attribute    Add Basic WWW Authentication, the attributes
	                    are a colon separated username and password.
	    -P attribute    Add Basic Proxy Authentication, the attributes
	                    are a colon separated username and password.
	    -X proxy:port   Proxyserver and port number to use
	    -V              Print version number and exit
	    -k              Use HTTP KeepAlive feature
	    -d              Do not show percentiles served table.
	    -S              Do not show confidence estimators and warnings.
	    -g filename     Output collected data to gnuplot format file.
	    -e filename     Output CSV file with percentages served
	    -r              Don't exit on socket receive errors.
	    -h              Display usage information (this message)
	    -Z ciphersuite  Specify SSL/TLS cipher suite (See openssl ciphers)
	    -f protocol     Specify SSL/TLS protocol (SSL2, SSL3, TLS1, or ALL)

例如现在要模拟10次请求百度首页：

	[root@chenllcentos ~]# ab -n10 http://www.baidu.com/

如果要模拟并发呢？可以新增`-c`参数。例如模拟10个用户同时并发，一共请求10次百度首页：

	[root@chenllcentos ~]# ab -c10 -n10 http://www.baidu.com/

一个网站的访问高峰期是最考验服务器性能的，因此进行压力测试的关键是评估好高峰期的服务器性能。幸运的是，AB为我们提供了`-t`参数，用来模拟在给定时间内的并发行为。例如模拟100个用户并发请求，持续时间60秒，可通过如下命令：

	[root@chenllcentos ~]# ab -c100 -t60 http://www.baidu.com/

用户访问页面不单单请求静态页面，更多时候需要传递请求参数。如果是GET请求，直接在url末尾添加请求参数即可；如果是POST请求，可通过`-p file`的形式向AB传递POST请求参数。例如：

	[root@chenllcentos ~]# ab -p /data/post.txt http://www.yourdomain.com/test.action

其中post.txt包含了POST请求参数：

	[root@chenllcentos ~]# cat /data/post.txt 
	{"phoneNumber":"18888888888","password":"123456"}