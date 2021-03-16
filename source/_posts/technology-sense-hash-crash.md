---
title: 技术场景之解决hash碰撞
cover:  /img/technology-sense/hash_crash_title.png
subtitle: 技术场景之解决hash碰撞
categories: "技术场景"
tags: "技术场景"
author:
nick: 袁苏东
link: https://github.com/yuansudong
typora-root-url: images
---

公司基于MONGODB+NFS建立了一套分布式文件存储系统,通过磁动态挂载磁盘的方式,达到动态扩容的目的.



该分布式文件存储系统通过外接CDN(腾讯,阿里,七牛云等)进行流量分发.其储存的主要类型有APK,EXE,IPA,EXCLE,PDF,各种类型的图片.



文件的上传通过HTTP上传,TOKEN鉴权的方式上传到服务器,再经过NFS网络共享到各个服务器结点上.



其中MONGODB的主要主作用是动态扩容+存储文件的HASH值.



通过计算文件的HASH值,上传服务结点可以通过HASH值确定该文件是否在NFS分布式存储系统中存在.



倘若存在,则直接返回URI.倘若不存在,则进行文件保存操作.



第一个版本,对于文件的HASH值,是通过MD5的方式计算出来的.以md5为唯一键与文件的URI映射上.



最初,想法是美好的.但是,我遇见了一个狗日的测试,他竟然跑到了国外的网站.下载了一个MD5碰撞的例子.



该网站如下,拿走,不谢!



```
http://www.win.tue.nl/hashclash/
```



结果是不言而喻的,我被狠狠地打了脸,只能灰头土脑的回来研究下HASH碰撞.



重新声明: 虽然说测试是狗日的,但是不得不承认别人的有些以及专业.狗日二字纯属戏言.



## 1.HASH简介



哈希(HASH)算法,即散列函数。它是一种单向密码体制,即它是一个从明文到密文的不可逆的映射，只有加密过程，没有解密过程.



同时,哈希函数可以将任意长度的输入经过变化以后得到固定长度的输出.



哈希函数的这种单向特征和输出数据长度固定的特征使得它可以生成消息或者数据.



## 2.HASH用途



### 01.数据检验



在安全接口开发中,在对请求参数进行处理之前,我们需要对请求体进行MD5或者SHA1验证.



当服务器计算出的SHA1或者MD5,与客户端计算的值一致时,就代表该请求参数,在传输途中没有被其他中间人修改过.



### 02.唯一标识



比如,有上千万个文件, 给你一个文件, 要你在这上千万个文件中查找是否存在.



一个很笨的办法就是把每一文件都拿出来, 然后按照二进制串一一进行对比. 但是这个操作注定是比较费时的.



而通用的方式是,可以用哈希算法对文件进行计算, 然后比较哈希值是否相同. 



### 03.HASH表



HASH表对于如今的程序员而言,已经是如雷贯耳了.这里就不再赘述了.



### 04.负载均衡



比如,现在又多台服务器, 来了一个请求, 如何确定这个请求应该路由到哪个路由器呢?当然, 必须确保相同的请求经过路由到达同一个服务器. 



一种办法就是保存一张路由关系的表, 比如用户的UID, 但是如果客户端很多, 势必查找的时间会很长. 这时, 可以将客户端的唯一标识信息(如:IP、username等)进行哈希计算, 然后与服务器个数取模, 得到的就是服务器的编号.



### 05.分布式存储



当我们有大量数据时, 一般会选择将数据存储到多个服务器, 为了提高读取与写入的速度嘛. 决定将文件存储到哪台服务器, 就可以通过哈希算法取模的操作来得到.



## 3.HASH函数



常见的HASH函数有,MD5,SHA1,SHA256,SHA512.通过windows的certutil 可以初步感受.



### 01.MD5



```
C:\Users\web>certutil -hashfile ./.gitconfig md5
MD5 的 ./.gitconfig 哈希:
38213d497c6bf32cfb3a67cb539cf8a3
CertUtil: -hashfile 命令成功完成。
```



### 02.SHA1



```
C:\Users\web>certutil -hashfile ./.gitconfig sha1
SHA1 的 ./.gitconfig 哈希:
92779fdfbc1200e396839a04e9713f6e12c151b3
CertUtil: -hashfile 命令成功完成。
```



### 03.SHA256



```
C:\Users\web>certutil -hashfile ./.gitconfig sha256
SHA256 的 ./.gitconfig 哈希:
4455f3d3ae7698f759a34f8c17290fa6664bd367157fd16daa5bfa5f54710aea
```



### 04.SHA512



```cmd
C:\Users\web>certutil -hashfile ./.bash_history sha512
SHA512 的 ./.bash_history 哈希:
7a1a14ef37e22ae24a1b04b397c8bd60c59567933b0d04776c5bec5486ac2aa39fc56eda5d6e2e5c52bd3484a3fdd51ec29accfe5dde3071f0cacc43eff2b159
CertUtil: -hashfile 命令成功完成。
```



 最终结论



通过调研,MD5和SHA1都有碰撞的风险,且互联网上已经有相关的举例.而SHA256在理论上有碰撞的可能.但是,目前还未实现.



最初的想法是,MD5和SHA1作为唯一键.但是再调研以及对比长度后,选择了SHA256