---
title: Hello World
subtitle: 人生若如初见,何时秋风悲画扇,等闲却道故人心,却道故人心意变
categories: "技术场景"
tags: "技术场景"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: hello-world
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)



```sequence
A --> B: hello
```





人生若如插件

```mermaid
graph TD;
    1[protoc解析处理] --> 2[拷贝经过protbuf序列化之后的二进制到标准输出]
    2 --> 3[自定义插件 protoc-gen-template]
    3 --> 4[从标准输出读取protobuf的二进制数据]
    4 --> 5[通过protobuf相关的库进行反序列化,比如,proto.Unmarshal]
    5 --> 6[反序列化之后,会获得plugin.CodeGeneratorRequest]    
```




![image-20210315203750702](/image-20210315203750702.png)