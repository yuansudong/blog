---
title: 企业网络之部署DNS
cover:  /img/network-deploy/dns_title.png
subtitle: 企业网络之部署DNS
categories: "企业网络"
tags: "企业网络"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: network-deploy-dns
---
### 01.通过yum源安装bind服务



```shell
yum -y install bind bind-utils bind-chroot
```



### 02.修改/etc/named.conf



```shell
options {        
	listen-on port 53 { any; }; // 此处要替换为any        
	listen-on-v6 port 53 { ::1; };        
	directory       "/var/named"; // 域名目录        
	dump-file       "/var/named/data/cache_dump.db";        
	statistics-file "/var/named/data/named_stats.txt";        
	memstatistics-file "/var/named/data/named_mem_stats.txt";        
	secroots-file   "/var/named/data/named.secroots";        
	recursing-file  "/var/named/data/named.recursing";        
	allow-query     { any; };  // 需要替换为any        
	forwarders      { 180.76.76.76; }; // 此处替换为一个公共的dns,比如百度或者阿里        
	recursion yes;        
	dnssec-enable no; // 需要和值为no,否则查询会很慢        
	dnssec-validation no; // 需要设置为no,否则查询会很慢        
	managed-keys-directory "/var/named/dynamic";        
	pid-file "/run/named/named.pid";        
	session-keyfile "/run/named/session.key";        
	include "/etc/crypto-policies/back-ends/bind.config";};
	logging {        
		channel default_debug {
        	file "data/named.run"; // 日志文件                
        	severity dynamic;        
        	};
     };
     zone "." IN {        
     	type hint;        
     	file "named.ca";
     };
     include "/etc/named.rfc1912.zones";
     include "/etc/named.root.key";
```



### 03.增加域名



此时,需要在/etc/named.rfc1912.zones下,增加公司的域名.此处以hfdy.com为例

```shell
zone "hfdy.com" IN {     
	type master;        
	file "hfdy.com.zone";
};
```



### 04.配置域名



```
touch /var/named/hfdy.com.zonevim /var/named/hfdy.com.zone# 将下列内容复制进named.hfdy文件中
```

```
$ORIGIN hfdy.com.$TTL 600        ; 10 minutes@    IN SOA dns.hfdy.com. dnsadmin.hfdy.com. (                                                2020060230  ;serial 每次更改都需要更改这个时间                                                10800    ;refresh 每三个小时刷新一次                                                900    ;每15分钟重试一次                                                604800      ;1周过期                                                86400    ;1 天更新                                               )                                             NS    dns.hfdy.com.$TTL  60   ; 1分钟过期
www            A     192.168.0.198
dns            A     192.168.0.198
redis          A     192.168.0.198
git            A     192.168.0.198
hub            A     192.168.0.198
file           A     192.168.0.198
sql            A     192.168.0.198
es01           A     192.168.0.198
ftp            A     192.168.0.198
cloud          A     192.168.0.198
pay            A     192.168.0.198
auth           A     192.168.0.198
account        A     192.168.0.198
mysql-dev      A     192.168.0.198
mysql-models   A     192.168.0.198
doc            A     192.168.0.198
sui            A     192.168.0.198
sed            A     192.168.0.198
nodes          A     192.168.0.198
chess          A     192.168.0.198
email          A     192.168.0.198
im             A     192.168.0.198
grpc           A     192.168.0.198
gid            A     192.168.0.198
api            A     192.168.0.198
jks            A     192.168.0.198
```



### 05.重启+开机启动



```shell
sudo systemctl restart named
sudo systemctl enable named
```