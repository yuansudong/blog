---
title: 企业网络之部署DOCKER
cover:  /img/network-deploy/docker_title.png
subtitle: 企业网络之部署DOCKER
categories: "企业网络"
tags: "企业网络"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: network-deploy-docker

---
Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的LINUX或WINDOWS机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口



### 01.安装docker-ce



```
curl https://download.docker.com/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
yum install https://download.docker.com/linux/fedora/30/x86_64/stable/Packages/containerd.io-1.2.6-3.3.fc30.x86_64.rpm -y
yum install docker-ce docker-ce-cli -y
systemctl start docker
systemctl enable docker
```



### 02.安装docker-compose



```
curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose --version
```



#### 03.配置daemon.json



```


cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m"
    },
    "insecure-registries":["192.168.42.128"],
    "storage-driver": "overlay2",
    "registry-mirrors":[
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ]
}
EOF
systemctl daemon-reload 
systemctl restart docker
systemctl enable docker
```

