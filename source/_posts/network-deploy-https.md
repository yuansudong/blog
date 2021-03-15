---
title: 企业网络之内网安全HTTPS
cover:  /img/network-deploy/https_title.png
subtitle: 企业网络之内网安全HTTPS
categories: "企业网络"
tags: "企业网络"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: network-deploy-https
---
HTTPS （全称：Hyper Text Transfer Protocol over SecureSocket Layer），是以安全为目标的 HTTP 通道，在HTTP的基础上通过传输加密和身份认证保证了传输过程的安全性



### 1.为什么需要在内网部署安全的HTTPS



(1) 保持正式环境与开发环境一致



(2) HTTPS普及,倘若不使用可信任的HTTPS,需要做一些额外的配置,比如DOCKER,GRPC,K8S等一些列应用.



(3) 方便一些语言做良好的开发环境,比如,GO中的gomod.



### 2.如何在内网部署被客户端信任的HTTPS



#### 01.Github项目



```
https://github.com/FiloSottile/mkcert
```



mkcert 是一个使用GO语言编写的生成本地自签证书的小程序，具有跨平台，使用简单，支持多域名，自动信任CA等一系列方便的特性可供本地开发时快速创建HTTPS环境使用.



#### 02.下载mkcert的可执行文件



```
https://github.com/FiloSottile/mkcert/releases/tag/v1.4.3
```



根据对应的系统,下载对应的二进制文件.



下载完毕之后,请将这二进制文件重命名为mkcert,并给mkcert可执行权限.



#### 03.生成CA根证书



在生成CA根证书之前,需要准备一台电脑.这台电脑的作用之一就是以后负责生成HTTPS证书的电脑.



因为个人喜好的原因.自从出道以来,用的服务器都是centos.所以这个HTTPS的生成的计算机系统采用centos8



```
yum install nss-tools -y./mkcert -install
```



#### 04.保存CA根证书



```
# 查看根证书的目录
[root@localhost https]# ./mkcert -CAROOT/root/.local/share/mkcert
[root@localhost https]# ll 
/root/.local/share/mkcerttotal 8-r--------. 1 root root 2484 Dec 24 21:33 
rootCA-key.pem-rw-r--r--. 1 root root 1688 Dec 24 21:33 rootCA.pem
```



将rootCA-key.pem rootCA.pem 保存妥善. 因为,每一台计算机都需要安装这个根证书.



个人一般是将其放在网络共享盘和基于NGINX搭建的文件服务器里.



因为,这样方便别人下载安装.



#### 05.生成HTTPS证书



```
[root@localhost https]# ./mkcert www.hfdy.com *.hfdy.com
Created a new certificate valid for the following names 📜 - "www.hfdy.com" - "*.hfdy.com"Reminder: X.509 wildcards only go one level deep, so this won't match a.b.hfdy.com ℹ️ The certificate is at "./www.hfdy.com+1.pem" and the key at "./www.hfdy.com+1-key.pem" ✅It will expire on 24 March 2023 🗓
[root@localhost https]# ls mkcert  
www.hfdy.com+1-key.pem  www.hfdy.com+1.pem
[root@localhost https]# 
```



## 3.WINDOWS安装CA根证书



#### 01.下载mkcert



```
https://github.com/FiloSottile/mkcert/releases/tag/v1.4.3
```



在上列地址中下载windows版本的可执行文件.并将mkcert-v1.4.3-windows-amd64.exe重命名为mkcert.exe



#### 02.创建CA证书目录



```
# 执行下列命令,不能用powershell打开.# 执行下列命令需要有管理员权限的cmdmkcert -install
```



#### 03.安装CA证书



```
E:\code\golang\bin>mkcert -CAROOTC:\Users\Administrator\AppData\Local\mkcertE:\code\golang\bin>
```



将证书服务器的rootCA-key.pem和rootCA.pem复制到本地. 并且用这两个证书替换掉mkert -CAROOT下的rootCA-key.pem和rootCA.pem



替换完毕后,再度执行下列命令



```
mkcert -install
```



至此,WINDOWS上的就配置完毕.



### 4.LINUX安装CA根证书



#### 01.下载mkcert



```
https://github.com/FiloSottile/mkcert/releases/tag/v1.4.3
```



在上列地址中下载linux版本的文件.并将mkcert-v1.4.3-linux-amd64.exe重命名为mkcert. 并给与可执行权限.



#### 02.创建CA证书目录

```
yum install nss-tools -y# 执行下列命令,不能用powershell打开.# 执行下列命令需要有管理员权限的cmd
mkcert -install
```



#### 03.安装CA证书



```
[root@localhost https]# ./mkcert -install
The local CA is already installed in the system trust store! 👍
The local CA is already installed in the Firefox and/or Chrome/Chromium trust store! 👍
[root@localhost https]# ./mkcert -CAROOT/root/.local/share/mkcert
```



将证书服务器的rootCA-key.pem和rootCA.pem复制或者下载到本地. 并且用这两个证书替换掉mkert -CAROOT下的rootCA-key.pem和rootCA.pem



替换完毕后,再度执行下列命令



```
mkcert -install
```



至此,LINUX上的就配置完毕.





### 5.MAC安装CA根证书



#### 01.下载mkcert



```
brew install mkcertbrew install nss
```



#### 02.创建CA证书目录

```
mkcert -install
```



#### 03.安装CA证书



```
mkcert -installmkcert -CAROOT
```



将证书服务器的rootCA-key.pem和rootCA.pem复制或者下载到本地. 并且用这两个证书替换掉mkert -CAROOT下的rootCA-key.pem和rootCA.pem



替换完毕后,再度执行下列命令



```
mkcert -install
```



至此,MAC上的就配置完毕.