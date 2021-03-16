---
title: 企业网络之部署GOGS服务
cover:  /img/network-deploy/git_title.png
subtitle: 企业网络之部署GOGS服务
categories: "企业网络"
tags: "企业网络"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: network-deploy-git
---


在进行部署之前,已经部署好了下面的这些环境



[企业网络之局域网安全HTTPS](https://www.yuansudong.top/2021/network-deploy-https/index.html)

[企业网络之部署DOCKER](https://www.yuansudong.top/2021/network-deploy-docker/index.html)

[企业网络之部署DNS服务](https://www.yuansudong.top/2021/network-deploy-dns/index.html)

[企业网络之部署NGINX](https://www.yuansudong.top/2021/network-deploy-nginx/index.html)



#### 01.创建GIT用户,并将其加入docker组



```

# 创建git用户
sudo useradd -s /bin/bash -d /home/git -m git
# 增加git用户入docker组
sudo gpasswd -a git docker
# 查看用户d
id git
uid=1000(git) gid=1000(git) groups=1000(git),984(docker)
```



#### 02.登录GIT,部署服务



```
su git
mkdir docker
cat > docker/docker-compose.yml <<EOF
version: "3.8"
services:
  gogs.hfdy.net:
    image: gogs/gogs:latest
    restart: always
    container_name: gogs.hfdy.net
    ports:
         - "10022:22"
         - "3000:3000"
    environment:
         - PUID=1000  # 此处的PUID 要和id git 命令的相同
         - PGID=1000  # 此处的PGID 要和id git 命令的相同
    volumes:
         - /home/git:/data
         
EOF

cd docker && docker-compose up -d --build
```



#### 03.配置NGINX反向代理



```
cat > /etc/nginx/hosts/gogs.hfdy.com.conf <<EOF
upstream backend.gogs  {
    server 127.0.0.1:3000 max_fails=0 fail_timeout=0s; #api 服务器 失败10次断60秒
    # server 127.0.0.1:8001 max_fails=10 fail_timeout=60s; #api 服务器 失败10次断60秒
    # server 127.0.0.1:8002 backup; # api 服务器 表示服务器为备用
    keepalive 64;   # 连接池的数量,不易太多
}
server {
    listen 80; #监听端口
    server_name gogs.hfdy.com; #域名
    rewrite ^(.*)$  https://$host$1 permanent; # 将http强制转换为https
}
server {
    listen       443 ssl ;
    server_name  gogs.hfdy.com;
    ssl_certificate /etc/nginx/cert/hfdy.pem;
    ssl_certificate_key /etc/nginx/cert/hfdy-key.pem; 
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1440m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "ECDHE-ECDSA-CHACHA20-POLY1305 ECDHE-RSA-CHACHA20-POLY1305 ECDHE-ECDSA-AES128-GCM-SHA256 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-RSA-AES256-GCM-SHA384 DHE-RSA-AES128-GCM-SHA256 DHE-RSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-SHA256 ECDHE-RSA-AES128-SHA256 ECDHE-ECDSA-AES128-SHA ECDHE-RSA-AES256-SHA384 ECDHE-RSA-AES128-SHA ECDHE-ECDSA-AES256-SHA384 ECDHE-ECDSA-AES256-SHA ECDHE-RSA-AES256-SHA DHE-RSA-AES128-SHA256 DHE-RSA-AES128-SHA DHE-RSA-AES256-SHA256 DHE-RSA-AES256-SHA ECDHE-ECDSA-DES-CBC3-SHA ECDHE-RSA-DES-CBC3-SHA EDH-RSA-DES-CBC3-SHA AES128-GCM-SHA256 AES256-GCM-SHA384 AES128-SHA256 AES256-SHA256 AES128-SHA AES256-SHA DES-CBC3-SHA !DSS";
    access_log  /nginx/logs/gogs.hfdy.com.access.log  main;      #访问日志路径 日志级别
    error_log   /nginx/logs/gogs.hfdy.com.error.log;   # 错误日志  # 可以在下方直接使用 [ debug | info | notice | warn | error | crit ]  参数
    location / {
        add_header Access-Control-Allow-Origin $http_origin;
        add_header Access-Control-Allow-Methods 'GET,POST,PUT,DELETE,FETCH';
        add_header Access-Control-Allow-Credentials true;
        add_header Access-Control-Allow-Headers 'x-mac,x-crfs-token,x-crfs-token,content-type';
        #限速处理,现在先不配置限速
        if ($request_method = 'OPTIONS' ) {
                return 204;
        }
        proxy_pass   http://backend.gogs/;
        #Proxy Settings
        proxy_redirect     off;    #关闭重定向
        proxy_set_header   Host             $host;  #在头部中设置host
        proxy_set_header   X-Real-IP        $remote_addr;  #在头部中设置请求的真是IP
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_request_buffering on; #接受完整的数据包之后再向上游转发
        proxy_connect_timeout      75;  #连接超时限制,默认为60s, 超时会返回502
        proxy_send_timeout         75; #发送超时时间
        proxy_read_timeout         75; #读取超时时间,默认为60s
        proxy_buffering on; #接受完整的上游响应,再发送给浏览器
        proxy_buffer_size          32k; #缓冲区大小
        proxy_buffers              16 64k; #响应的缓冲区大小
        proxy_busy_buffers_size    64K; #繁忙的缓冲区大小
        proxy_temp_file_write_size 64k; #中间文件的写大小
        proxy_max_temp_file_size 1024m;   #响应的最大中间文件,默认就是1G
        proxy_limit_rate 0; #不限制读取上游的响应速度
        proxy_http_version 1.1; #http的版本
        proxy_set_header Connection ""; #设置connect
    }
}
EOF

nginx -t
systemctl reload nginx
```



#### 04.打开浏览器配置GOGS


![图片](/acc.png)


配置好之后,基本可以使用



#### 05.docker与宿主机共享22端口



```
su git
ssh-keygen -t rsa -b 4096 -C "git@gogs.hfdy.com"
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
# 在 authorized_keys 最前面添加下面的
# no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty
# example:no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC0QmMG+Xz+qjIEp7Vrlt5SLGergYF4DJbY7+4l7R9nvhrbbSSVzNiKGngSCPvQWfkLv1XZhlzG9V7Pr/Gj86yJcz4/AvMYkZanVZ+zi9JFb3CfJLtxeMSUgNC6Ig2UDxAW53hGpADlFL+4Ic9yMoNw0AoqmU2FfKjfO3kFc8xXJ+vAboXgEukxWl/3vniMzIwDiQAfTftbV9RUl2UmbHO70l65FrIvo5llagvSSzu9q3xqhVN5CHdOvRFOEnNJ/xdvnoUE7hbYTjoooOK5o6paXtbPkNWlp3vpyjXnMAxJOYMr+0Gp+S1hGoTMnLyTE/94KKvC7wgrpLpKkd9CnD2uT7B/60tj+tn+hQnBTB5dVqQ1eWUhQ7tySqXXyDuWzAin5CI9q+FlCoK83OAKlFJMopIVgESf19xLkD/x064oyIY0X3q9gZ7QeZk5QSI5vIDFdG9+SYyUUhwYxSnuPn6fmn1RbetOWguzBXy6betP4jQE7HxU4RC/6r+TxapdXSmDfQ92Pjd9Qx9atJ6ntjV6bcxU3u7j5XijaDhkoIPbK5cqnTm7OXZjjfkhWeNAI+IkM2ubF6Ya53j3eOHqKE64h5aIOWgkxjojoxf4spkrGfIeE5aD6Ab4+9Lyu0KmBsCzuTT66oS2gQlQcQ/vmcIgOrGjMDd07GfAjURUzsmgxw== git@gogs.hfdy.com
```



#### 06.配置GIT用户SSH登录



```
[root@localhost ~]$ mkdir -p /app/gogs/
[root@localhost ~]$ cat >/app/gogs/gogs <<'END'
#!/bin/sh
ssh -p 10022 -o StrictHostKeyChecking=no git@127.0.0.1 \
"SSH_ORIGINAL_COMMAND=\"$SSH_ORIGINAL_COMMAND\" $0 $@"
END
[root@localhost ~]$ chmod 755 /app/gogs/gogs
```



#### 07.遇见的问题



  Q: git clone 在windows上报SSL certificate problem: unable to get local issuer certificate.



  A: 对于mkcert生成的证书以及安装,于ubuntu下是不会有这个问题的.在windows下需要使用如下命令解决.

  

```
git config --global http.sslverify false
```