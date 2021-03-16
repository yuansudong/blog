---
title: 技术场景之解决不知线上代码版本的问题
cover:  /img/technology-sense/code_version_title.png
subtitle: 技术场景之解决不知线上代码版本的问题
categories: "技术场景"
tags: "技术场景"
author:
    nick: 袁苏东
    link: https://github.com/yuansudong
typora-root-url: images
---

GIT是一个分布式版本控制软件，最初由林纳斯·托瓦兹创作，于2005年以GPL发布。最初目的是为更好地管理Linux内核开发而设计。最终走向了全世界



在以前的工作中,GIT常被用作代码管理,以及版本控制,具有回滚到指定的提交点的功能.



## 1.问题场景



每当版本发布的前一周,是最忙的日子.各种各样的问题频繁地爆出,而让我最难以接受的是,GOLANG编译之后的二进制执行文件发布到线上之后,不知道这个二进制可执行文件到底包含了哪些功能.



以致于会听到,诸如404等错误码的反馈.当听到这些反馈之后,我看着这个二进制执行文件,不知是错误码返回错误,还是在编译的时候没从开发分支上集成这块的功能.



亦或者是编译时,编译错了分支,错把开发分支编译成了产品分支,最终发布到了线上.



亦或者是客户端的接口接入的有问题



亦或者是编译时,编译错了版本,等等情况.



仅仅看着一个二进制的文件,按照我现在的水平,我看不出里面有什么东西.



一般出现这个错误之后,就会把GIT打开,再去拉取指定版本的代码将其重新编译,发布至线上.



倘若能够第二次成功也好,但最害怕的是还有第三次,第四次,第N次.



而线上,是能够让你试一试的地方么? 答案肯定是不能.



## 2.解决办法



在编译时,将提交的ID编译进golang的二进制程序中,通过一番调研,发现GO在编译时,可通过下面的命令办到.



```shell
go build -ldflags "-X main._GitBranch=${branch} ..." -o appname
```



main为包名,_GitBranch为main包下的一个变量.



那么,到底需要将哪些元素,在编译时指定呢?根据个人实践会有所不同,而我却打入了下列的元素



App Name: AppName用于指定该应用的名称,为了防止运维将二进制文件改名,导致问题定位错误



App Version: 当前应用的版本,也就是git tag上的版本,为了防止在发布到线上时编译错了版本,导致需要发布的新功能未能发布到线上.



Git Branch: 编译时所编译的分支,用于查看是否将开发分支发布到了线上.



Git Commit:最后一次的提交ID,通过该ID,我们可以通过git log排查功能在从开发分支合并到产品分支时,到底有没有漏掉



Go Version:编译时golang的版本,尽管关于golang版本带来的问题不多,但是需要打上,防止以为golang版本所产生的错误,这种错误,非常难查.所以,需要标记



Build System: 在编译时,编译所在的操作系统类型.众所周知,GO有交叉编译,可在windows上编译linux,mac的可执行文件.而我被这种交叉编译坑的一次是在压测的时候,该压测的环境在centos上,但是我却在windows上编译了linux的可执行文件.最终在发生OOM时,通过交叉编译的可执行程序没有打印问题,而通过centos上编译的二进制文件输出了信息.



Build Time: 编译时的时间,该时间主要是为了印证"人无完人"这句话,一般情况下不会有问题,但是倘若在上线时,同个版本下,频繁更新,谁知道进程有没有更新呢.此时,可以通过编译时间



### 01.一般流程



我在发布版本时,会从开发分支上通过提交日志,辅助以下列命令,将一个个的功能提交合并到产品分支上.



```sh
git cherry pick [commitId]
```



commitId 可以通过 git log 命令查看,查看时会显示提交的信息.根据信息去合并功能.



当所有功能全部合成到产品分支上之后,会进行提交,提交会产生一个commitId,



基于该commitId,通过打tag的方式,可以向该时git上产品分支的状态进行备份



```shell
git tag v0.1 -m "comments.." [commitId]
```



通过该备份,可以随时随地的进行还原.



最后可通过,jenkins自动打包,k8s自动部署.



### 02.linux编译脚本



```shell
#!/bin/bash
os=$(go env GOOS)
arch=$(go env GOARCH)
goversion=$(go version | awk '{print $3}')
commitid=$(git rev-parse --short HEAD)
account=$(git log --pretty=format:"%%an" -1)
branch=$(git branch --show-current)
nowtime=$(date +%Y-%m-%d.%H:%M:%S)
appversion=1.6
appname=example
go build -ldflags "-X main._GitBranch=${branch} -X main._AppName=${appname} -X main._AppVersion=${appversion} -X main._OS=${os} -X main._Arch=${arch} -X main._GoVersion=${goversion} -X main._GitCommit=${commitid} -X main._GitAccount=${account} -X main._DateTime=${nowtime}" -o ${appname}
```



输出结果



```shell
[root@localhost cmdline]# ./example version
App Name     :    example
App Version  :    1.6
Git Branch   :    master
Git Commit   :    ff72c3a
Git Account  :    yuansudong
Go Version   :    go1.14.12
Build System :    linux
Build Time   :    2021-03-11.03:02:36
Build Arch   :    amd64
```



### 03.windows编译脚本



```bash
@echo off
for /F %%i in ('go env GOOS') do ( set os=%%i)
for /F %%i in ('go env GOARCH') do ( set arch=%%i)
for /F %%i in ('go env GOVERSION') do ( set goversion=%%i)
for /F %%i in ('git rev-parse --short HEAD') do ( set commitid=%%i)
for /F %%i in ('git log --pretty^=format:"%%an" -1') do ( set account=%%i)
for /F "tokens=* delims=" %%i in ('git branch --show-current') do ( set branch=%%i)
for /f "tokens=* delims=" %%i in ('echo %date:~0,4%-%date:~5,2%-%date:~8,2%.%time:~1,1%:%time:~3,2%:%time:~6,2%') do ( set nowtime="%%i")
set appversion=1.6
set appname=example
go build -ldflags "-X main._GitBranch=%branch% -X main._AppName=%appname% -X main._AppVersion=%appversion% -X main._OS=%os% -X main._Arch=%arch% -X main._GoVersion=%goversion% -X main._GitCommit=%commitid% -X main._GitAccount=%account% -X main._DateTime=%nowtime%" -o %appname%.exe
```



输出结果



```cmd
PS E:\code\golang\src\github.com\yuansudong\cobra\generator\cmdline> .\example.exe version
App Name     :    example
App Version  :    1.6
Git Branch   :    master
Git Commit   :    d3b1940
Git Account  :    yuansudong
Go Version   :    go1.16
Build System :    windows
Build Time   :    2021-03-10.6:52:52
Build Arch   :    amd64
PS E:\code\golang\src\github.com\yuansudong\cobra\generator\cmdline>
```