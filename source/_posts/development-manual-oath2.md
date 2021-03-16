---

title: 开发手册之oath2,以google api为例
cover: /img/development-manual/oath2_title.png
subtitle: 开发手册之oath2,以google api为例
categories: "开发手册"
tags: "开发手册"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-oath2
---

OATH2.0,一个不同应用间的授权机制. 其核心是,用户使用一个应用,这个应用需要这个用户的一些资源.



而这个应用要使用的用户的资源,却不是这个应用能提供的,属于三方.此时,这个应用需要向三方发送请求,去拿到这个用户的资源.



但是,由于安全,三方服务,需要确保这个资源是用户同意这个应用获取的.



这一些列,确保的过程,就是OATH2.0



## 1.应用场景



新开发的应用,随着系统业务的扩展,要慢慢地融入互联网.



在互联网中,你有用户或者资源.那么,新开发的应用就可以和你合作,使用你的资源或者用户,亦或者是名气.达到快速引流推广的目的. 比如,QQ,微信,微博,GITHUB,GOOGLE,FACEBOOK,TWITTER等



随着应用逐渐成熟,积累了大批用户,积累了大批的用户资源. 此时,这个应用便可以效仿QQ等应用,让其他正在孵化的项目接入自己的平台. 让这个孵化项目,使用自身的资源,达到快速成长的目的.



随着用户越来越多,业务越来越庞杂,原本的单体服务,变得越来越庞大,常常遇见"牵一发而动全身"的场景.



此时,需要根据业务或者职业划分,将原本的单体服务,拆解成不同的服务,期望达到各司其职的目的.



上面的这些场景,通过网络触发,任何设备都可以通过网络发起请求.但是,哪台设备能成功拿到数据,这就需要一套认证机制.



而OATH2.0,也就应运而生.



## 2.授权流程



![图片](/title.png)

1.  用户使用应用.
2.  应用向用户发起授权许可.
3.  用户同意授权,发起请求至三方授权服务器.
4.  三方授权服务,验证信息,合法.
5.  三方授权服务器返回访问令牌.
6.  应用携带令牌向三方资源服务器请求资源.
7.  三方资源服务器验证访问令牌,合法.
8.  返回资源给应用.
9.  应用向用户呈现资源或者缓存.

## 3.授权方式



OATH2.0的标准是在RFC6749文件中定义,可通过下列地址访问



```
https://tools.ietf.org/html/rfc6749
```



在该文件中,定义了四种授权流程,他们分别是,授权码式,隐藏式,密码式,客户端凭据式.



以上四种方式,在申请授权,获得访问令牌时,都需要在三方应用的后台进行备份申请客户端的ID和客户端的密钥.



通过客户端的ID和客户端的密钥,可以标识申请授权的应用,以及申请访问令牌的安全性.



### 01.授权码式



该种方式,是指应用在获取三方资源的访问令牌前,首先需要向三方授权服务申请一个授权码(Authorization Code). 通过该授权码,获得访问令牌.



### 02.隐藏式



该种方式也被称之为简化式,是指直接向前端应用颁发令牌.因为这种获得访问令牌时,没有授权码的流程.因此,被称为隐藏式或者简化式.



### 03.密码式



该种方式是指用户直接将自己三方的用户名和密码告诉给应用,应用通过用户名和密码,通过HTPP请求,向三方服务器,直接申请访问令牌.



### 04.凭据式



该种方式是指,用户在三方的后台申请客户端ID以及客户端密钥.在申请访问令牌时,HTTP请求参数中带上客户端ID和客户端密钥.



## 4.更新令牌



在OATH2.0中,访问的令牌是有时效性的,在经历过一段时间后,令牌就会被置为不可用,一旦不可用,三方资源服务器就不会再提供服务.



因此,API调用者需要根据访问令牌的过期时间去刷新令牌.



## 5.案例举例



下面,以GOOGLE API为例, 介绍下四种授权方式.此时,我们需要开发的应用时,通过用户授权,去拉取该用户名下Admob广告收益报表.



### 01.授权码式



授权码式,适合服务端渲染页面的应用,即SSR(Server Side Render).通过此种应用去拉取admob的广告收益报表时.



#### 1). 获取客户端ID和客户端的密钥



需要通过下列地址申请一个OAuth客户端ID



```
https://console.developers.google.com/apis/credentials
```



![图片](/2.png)

![图片](/3.png)



#### 2). 发起授权



可通过下面的HTTP请求发起授权



```
Host:https://accounts.google.com
Uri:o/oauth2/v2/auth
Method: GET
Paramters:  
   client_id:test_client_id # 在https://console.developers.google.com/apis/credentials 获取到的客户端ID 
   redirect_uri:https://google.hfdy.com/code #用户同意授权过后,要跳转的URL
   response_type:code # 对于 Server Side Web Application,此处固定为code,即授权码模式
   scope: https://www.googleapis.com/auth/cloud-platform #一个以空格分隔的字符串,意思是指,该应用可以有哪些权限
   access_type: online|offline # 用于表明,当用户不在浏览器时,应用是否可以刷新令牌,当用户不在浏览器中,需要刷新令牌时,该值要设置为offline
   state: anystring # 该参数的值会在授权成功后以 state=anystring出现在重定向的URI中
   include_granted_scopes: true|false # 该参数指定,应用是否可以动态申请权限.true代表可以
   login_hint: email或者google id # 如果应用程序知道是哪个用户在发起授权,可以在此处填上用户的邮箱或者google id.此种方式可以简化授权流程
   prompt: none|consent|select_account # 是否提示用户选择账户
```



发起授权时的页面如下



![图片](/1.png)



授权同意后,服务器会受到下面的请求



```
http://localhost/?state=my_args&code=4/0AY0e-g7kLX5rSEm2PEnoXIZq7lyekG1SbEzT1BSHvmS6SDpURwzc_MKhu1SCrrgomm9hOQ&scope=https://www.googleapis.com/auth/admob.report
```



其中code就是授权码.



#### 3). 根据授权码获取访问令牌



拿到授权码之后,可以根据下列的HTTP请求获得访问令牌



```yaml
Host:https://oauth2.googleapis.com
Uri:/token
Method:POST
Content-Type:application/x-www-form-urlencoded
Paramters:
  code: 4/0AY0e-g7kLX5rSEm2PEnoXIZq7lyekG1SbEzT1BSHvmS6SDpURwzc_MKhu1SCrrgomm9hOQ
  client_id:test_client_id # 客户端的ID
  client_secret:test_client_secrect # 客户端的密钥
  redirect_uri: https://google.hfdy.com/code
  grant_type:authorization_code # 固定为authorization_code
```



发起该请求,可以得到下列的JSON数据



```json
{
  "access_token": "1/fFAGRNJru1FTz70BzhT3Zg",
  "expires_in": 3920,
  "token_type": "Bearer",
  "scope": "https://www.googleapis.com/auth/admob.report",
  "refresh_token": "1//xEoDL4iW3cxlI7yDbSRFYNG01kVKM2C-259HOF2aQbI"
}
```



#### 4). 调用请求



访问三方资源服务器



```yaml
GET /drive/v2/files HTTP/1.1
Host: www.googleapis.com
Authorization: Bearer ya29.A0AfH6SMB8cWo8n9CTpa03BfAYOk_DQKNCQDfUlg2WCQO3wJeCP1rNe6_KlrwIW4g817zJgOfPb0_FT6RYbET60Lo-jVNa23XtVu3MOyQXVeaKGwiCskOWiUAS01yIerhYE0l6e7icTiHW_lC92Mr0jgNnFdg1
```



#### 5). 刷新token



通过下列的请求可以刷新token



```curl

curl --location --request POST 'https://oauth2.googleapis.com/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=57546962315-lqmjq7aescgc7eggogphaevk8h3jt3kk.apps.googleusercontent.com' \
--data-urlencode 'client_secret=SFT54d6vai6c_DbTBJctkAOX' \
--data-urlencode 'refresh_token=1//06O7KoZJqdX7yCgYIARAAGAYSNwF-L9IrtWpvERBTXDZV7f9TX1ZDJfUiURwqIez0FkUWxrcvcpC1q8-aod2ikz-VNTd30Q6eBtU' \
--data-urlencode 'grant_type=refresh_token'
```



响应体



```json
{
  "access_token": "1/fFAGRNJru1FTz70BzhT3Zg",
  "expires_in": 3920,
  "scope": "https://www.googleapis.com/auth/admob.report",
  "token_type": "Bearer"
}
```



6) 序列图



![图片](/7.png)



### 02.隐藏式



对于一些纯前端应用,在GOOGLE中被称之为JAVASCRIPT WEB APP. 这类应用获取访问令牌,减少了授权码的步骤.



#### 1) 发起授权,获取访问令牌



发起请求



```
curl --location --request GET 'https://accounts.google.com/o/oauth2/v2/auth?include_granted_scopes=true&scope=https://www.googleapis.com/auth/admob.report&response_type=token&state=state_parameter_passthrough_value&redirect_uri=https://google.hfdy.com/callback&client_id=691541517620-hp23tqgh6itpj278eqmm1nr2ndqt3eif.apps.googleusercontent.com'
```

重定向页面

```
https://google.hfdy.com/callback#access_token=ya29.A0AfH6SMC7H3A5DEHj3RqrjDxhOp7Ou9iin7lqgcrkvxRZEAwOcr-mCmlpCtv4pG-uWA7-zTx42XYM38sATuDXpx5-pb7hZx8uzTGzoMQAc6rtemh2hTogglfy6cZe39f8RHSyQDWM-qE62v15V0sLEWCLqhFW&token_type=Bearer&expires_in=3599&scope=https://www.googleapis.com/auth/admob.report
```



在重定向的URL中通过锚点的方式,将访问令牌放在了锚点中,JAVASCRIPT 应用可以通过该URL获得访问令牌



#### 2) 访问GOOGLE的资源服务



```
GET /drive/v2/files HTTP/1.1
Host: www.googleapis.com
Authorization: Bearer ya29.A0AfH6SMB8cWo8n9CTpa03BfAYOk_DQKNCQDfUlg2WCQO3wJeCP1rNe6_KlrwIW4g817zJgOfPb0_FT6RYbET60Lo-jVNa23XtVu3MOyQXVeaKGwiCskOWiUAS01yIerhYE0l6e7icTiHW_lC92Mr0jgNnFdg1
```



此种方式,没有刷新token的方式.



当token过期后,需要重新调用认证流程.此时,有参数可选择是否再度走认证流程.



#### 3) 序列图



![图片](/8.png)



### 03.密码式



目前GOOGLE-API不提供密码式去访问API



### 04.凭据式



可参考该链接: [开发手册之JWT,以GOOGLE为例](https://www.yuansudong.top/2021/development-manual-jwt/index.html)



参考连接



https://oauth.net/2/

https://tools.ietf.org/html/rfc6749

https://developers.google.com/identity/protocols/oauth2/web-server#httprest

