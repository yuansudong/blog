---
title: 技术场景之解决推广裂变异常
cover:  /img/technology-sense/fission_exception_title.png
subtitle: 技术场景之解决hash碰撞
categories: "技术场景"
tags: "技术场景"
author:
nick: 袁苏东
link: https://github.com/yuansudong
typora-root-url: technology-fission-exception
---

随着互联网裂变式自媒体的不断崛起,低成本引流成为企业的强烈需求.伴随着微信,FACEBOOK,微博,B站,头条等自媒体平台的崛起.



裂变式营销逐渐成为推广的一种方式.而其中"社交裂变"成为了其中的重中之重.



近年来,随着微信生态的发展与成熟,形成了具有中国特色的商业模式和形态.任何一家企业或者个人都可以通过微信的生态圈获得百万级,乃至千万级的用户.



而裂变营销的核心之一是确保推广人的利益.倘若,推广人的利益受损,又有哪个推广人愿意继续推广呢?



当然,不排除兴趣.但是,兴趣这个词太过于"玄妙".非正常人所能触及,更多的趋向于利益.



那么,要如何确保推广人的利益,是裂变的核心话题.



## 1.问题场景



用户在注册时,不愿意输入附加信息.而此类附加信息是缔造邀请人与被邀请人联系的核心,比如邀请码.



用户不愿意输入附加信息,程序就不要让用户输入附加信息.将附加信息,在注册时,以静默的方式放在注册接口里,上传给服务器



## 2.裂变流程



### 01.已知用户复制推广链接发给未知用户



推广链接举例:https://dl.ptm.com/invation.html?code=AAC98Z&channel=vivo



其中code便是种子用户的邀请码.channel便是推广渠道.



### 02.未知用户点击链接进入浏览器下载页面



在用户进入下载页面之后,该页面首先要做的是将当前系统的环境和邀请码以及渠道信息,通过接口上传至服务器.



目前,Javascript 可以拿到下面的信息



```yaml
AppVersion: 5.0 (windows nt 10.0; win64; x64) applewebkit/537.36 (khtml, like gecko) chrome/89.0.4356.6 safari/537.36
Language: zh-cn
AppCodeName: mozilla
AppName: netscape
Memory: 8G
CPU: 4核
屏幕分辨率: 1920 X 1080
GPU: angle (intel(r) hd graphics 4600 direct3d11 vs_5_0 ps_5_0)
```



相关代码如下



```javascript
$("#user_agent").text(window.navigator.appVersion.toLowerCase());
$("#language").text(window.navigator.language.toLowerCase());
$("#app_code_name").text(window.navigator.appCodeName.toLowerCase());
$("#app_name").text(window.navigator.appName.toLowerCase());
$("#memory").text(window.navigator.deviceMemory+"G");
$("#cpu").text(window.navigator.hardwareConcurrency+"核");
$("#width_height").text(window.screen.width*window.devicePixelRatio+" X "+window.screen.height*window.devicePixelRatio);
var canvas = document.createElement('canvas');
var gl = canvas.getContext('experimental-webgl');
var debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
$("#gpu").text(gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL).toLowerCase());
```



客户端需要上传的参数如下.



```yaml
channel: 渠道 # string 渠道未定,正规的就是那几大运营尝试.  https://dl.ptm.com/invation.html?code=AAC98Z&channel=vivo,可从连接中获取chnnael,code和channel不一定都有.
invationCode: AAC98Z #  string   https://dl.ptm.com/invation.html?code=AAC98Z&channel=vivo ,可从连接参数中获取
language: zh_cn # string 采用全部小写,需要用toLowerCase()转换.
cpu: 4 # number 
resolutionWidth: 1920 # number 屏幕分辨率中的宽
resolutionHeight: 1080 # number 屏幕分辨率中的高
gpu: angle (intel(r) hd graphics 4600 direct3d11 vs_5_0 ps_5_0) # string GPU的型号
```



调用该接口后,服务端会在后台数据库中生成一条记录,该记录将邀请码与IP以及系统环境信息绑定在一起.



并且服务端会返回,当前的系统信息[IOS,ANDROID,UNKNOWN],以及相关系统的下载连接



web端可通过动态创建a标签,模拟人为点击,进行下载.



```javascript
function copy(text){
        let textarea = document.createElement('textarea');
        textarea.id = "copyTextarea";
        textarea.style.width = 0;
        textarea.style.height = 0;
        document.body.appendChild(textarea);
        textarea = document.getElementById('copyTextarea');
        textarea.innerHTML = text;
        
        if( "android" == "ios"){ //  此处需要获取本机,判断是不是ios
            const range = document.createRange();
            range.selectNode(document.getElementById('copyTextarea'));
            const selection = window.getSelection();
            if (selection.rangeCount > 0) selection.removeAllRanges();
            selection.addRange(range);
        }else{
            textarea.select(); // 选中文本(select()方法对IOS部分版本无效)
        }
        document.execCommand('copy');
        document.body.removeChild(textarea);
 }
 $("#dl_btn").click(function(){
        copy("<huowu.ptm.com:invation>AAC98Z</huowu.ptm.com:invation>")
        var eleA = document.createElement('a');
        eleA.setAttribute("href","接口中返回的一个下载地址");
        elseA.click(); // 模拟人为点击,避免打开新的页面去下载 
 })
```



### 03.未知用户进入注册流程



为了方便提取,此处使用正则表达式的方式提取邀请码.正则表达式如下.



```
<huowu.ptm.com:invation>([\s\S]*?)</huowu.ptm.com:invation>
```



下面是GO代码举例,达到一个抛砖引玉的过程.



```go
/**********************************************************************
假设剪贴板中的内容是这样:
  start 
  <huowu.ptm.com:invation>ACD78Z</huowu.ptm.com:invation> 
  end 
  人生若如初见,何事秋风悲画扇
    等闲却道故人心,却道故人心易变
    忽有故人心上过,回首山河已是秋
    两处相思同沐雪,此生也算共白头
  <huowu.ptm.com:invation>一个旧的邀请码</huowu.ptm.com:invation>
***********************************************************************/

func main() {
    _GrapInvationCode()
}

// _GrapInvationCode 抓取
func _GrapInvationCode() {
    sCliboradContent = `
    start 
  <huowu.ptm.com:invation>ACD78Z</huowu.ptm.com:invation> 
  end 
  人生若如初见,何事秋风悲画扇
    等闲却道故人心,却道故人心易变
    忽有故人心上过,回首山河已是秋
    两处相思同沐雪,此生也算共白头
  <huowu.ptm.com:invation>一个旧的邀请码</huowu.ptm.com:invation>
    `
    reg, err := regexp.Compile("<huowu.ptm.com:invation>(.*?)</huowu.ptm.com:invation>")
  if err != nil {
    log.Println(err.Error())
    return
  }

  sMatchArr := reg.FindAllString(sCliboradContent, 1) // 1指,匹配到一个就返回.
  if len(matchArr) == 0 {
    return // 代表剪贴板中,无该格式的内容
  }
  sDstStr := sMatchArr[0]
  sInvationCode := reg.ReplaceAllString(sDstStr, "$1")
  log.Println(sInvationCode)
}
```



通过提取,会输出下面的内容



```
2021/01/25 10:08:44 ACD78Z # ACD78Z 就是该用户的最新邀请码
```



## 3.异常情况



### 01.客户端没有在剪贴板中获得邀请码



据搭档反馈,他每次只能拿到剪贴板的最新的一条数据.倘若未知用户的环境在注册app前,进行了复制操作.那么,剪贴板中的邀请码就会被清空.



客户端拿不到邀请码,服务端就无法让新注册用户与邀请人建立联系,直接导致邀请人利益受损.



关系好的,直接让你在后台修改. 关系不好的,直接终止推广.



针对此类的情况,采取的补救措施是在注册接口中,同时附带如下信息.



```yaml
language: zh_cn # 系统的默认语言环境 string 采用全部小写,需要用toLowerCase()转换.
cpu: 4 # number 
resolutionWidth: 1920 # number 屏幕分辨率中的宽,此处分辨率,需要web端和移动端对一下,有的可能拿到的是屏幕的宽度或者高度.而不是分辨率.以前有同时跳过这个坑
resolutionHeight: 1080 # number 屏幕分辨率中的高
gpu: angle (intel(r) hd graphics 4600 direct3d11 vs_5_0 ps_5_0) # string GPU的型号
system: "ios" # 此处有三个选项,[ios,android,unknown]
```



通过上面的参数,服务器这面会根据请求的IP,查出一个该IP下的所有下载记录.这些记录里包含着每次下载的邀请码.



根据每条参数里的信息,做一个分值匹配.取分值最高的作为新注册用户的邀请码.



### 02.客户端在剪贴板中没拿到邀请码.而多个下载用户处于同一局域网



处于同一局域网,意味着这些下载IP对于服务器而言是一致的.倘若是不同的邀请码.服务器则以该IP下最新的邀请码,作为新注册的邀请码.



### 03.客户端没有拿到邀请码.而该IP又没有下载记录.



对于此种情况,发生的场景一般是.邀请人直接下了个安装包.发在了他的资源群.资源群的未知用户,通过该安装包进行安装.



此种情况,导致了客户端拿不到邀请码.而服务端又没有未知用户的IP下载记录.



服务端处理的方式是,对于此类情形,将新注册的用户归纳为自然流入用户.即该新注册用户没有邀请人.