---

title: 基于hexo+github+自定义域名,搭建安全https的博客网站
cover: /img/document-tool/hexo_title.png
subtitle: 基于hexo+github+自定义域名,搭建安全https的博客网站
categories: "文档工具"
tags: "文档工具"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: document-tool-hexo
---

## 1.github

### 01.注册账号

## 地址：https://www.github.com

![image-20210317053248148](/image-20210317053248148.png)

### 02.建立博客仓库



博客仓库的建立是有要求的，其要求便是xxx.github.io。而xxx是你的个人名字或者组织名称，比如这里就是yuansudong。建立好之后，其格式如下。

![image-20210317053920237](/image-20210317053920237.png)



## 2.域名



### 01.注册



域名的注册，是为了自定义域名准备的。倘若你习惯以xxx.github.io访问自己的博客，可以跳过这一步。

而个人的DNS是在腾讯云注册的，吐槽一句，腾讯云的使用体验，真的是其他xxx不能比的，google除外。

地址：https://dnspod.cloud.tencent.com/?from=qcloudHpProductDns/

![image-20210317054230880](/image-20210317054230880.png)

个人挑了个最便宜的域名，以个人名称+top结尾，1元钱/首年。



### 02.解析记录



在域名解析完毕后，需要在DNSPOD那里添加两行解析记录，其解析记录的类型为CNAME,其目的是将自身域名解析成xxx.github.io.

地址：https://console.cloud.tencent.com/cns

![image-20210317054934953](/image-20210317054934953.png)

www 是为了解析www.yuansudong.top到yuansudong.github.io.

@是为了解析yuansudong.top到yuansudong.github.io

一定要加这两个解析，不然为自己的博客接入广告的时候，是个麻烦事情，比如Google Adsense



## 3.hexo搭建

### 01.安装node.js

地址：http://nodejs.cn/download/

![image-20210317055530921](/image-20210317055530921.png)



测试命令

```bash
node -v
```

为npm设置淘宝镜像

```bash
npm config set registry https://registry.npm.taobao.org
```

验证

```bash
npm config get registry 
```

输出结果

```bash
D:\>npm config get registry
https://registry.npm.taobao.org/
```

### 02.全局安装hexo

```
npm install -g hexo
```

### 03.在github上建立一个博客仓库

在github上建立一个博客仓库的目的,是为了保存自身博客不丢失.前面的xxx.github.io仓库只是用来发布时的仓库.我个人的仓库名称为blog.

![image-20210317061018866](/image-20210317061018866.png)

### 04.克隆仓库

#### 1)克隆仓库

```bash
git clone https://www.github.com/yuansudong/blog.git
```

#### 2)hexo初始化

hexo的初始化要求必须在一个空目录下,而blog仓库中,至少有.git文件夹.所以需要先建立一个文件夹之后再初始化

```bash
mkdir temp && cd temp && hexo init && npm install && hexo s -g	
```

![image-20210317061959427](/image-20210317061959427.png)

此时,你就可以通过在浏览器中打开http://localhost:4000进行对博客的访问了.我本地访问的页面如下

![image-20210317062108396](/image-20210317062108396.png)

### 05.美化hexo

hexo安装完毕之后,下一步的事情就是要进行博客的美化工作,毕竟最原始的hexo,很难满足自身的意愿.

hexo的主题地址可通过该链接进行访问:https://hexo.bootcss.com/themes/

![image-20210317062709087](/image-20210317062709087.png)

因为是演示文档,所以我随便选了一个主题,是https://github.com/Halyul/hexo-theme-mdui

在hexo的根目录下,本人是temp,运行下列命令:

```bash
cd themes && git clone https://github.com/Halyul/hexo-theme-matery.git
```



克隆完毕后,将hexo根目录(temp下)的_config.yml打开,修改主题为hexo-theme-mudi,并修改一些相关的配置.

![image-20210317070431369](/image-20210317070431369.png)



输入下列命令验证结果

```bash
hexo s -g
```

![image-20210317070525288](/image-20210317070525288.png)

至于其他的相关配置可以从github上的主页上查看.



## 搜索引擎收录

### 01.百度收录

#### 1）安装百度收录插件

```bash
npm install hexo-generator-baidu-sitemap --save
```



## 02.谷歌收录

#### 1） 安装谷歌收录插件

```bash
npm install hexo-generator-sitemap --save
```

