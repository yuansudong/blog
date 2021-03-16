---

title: 开发手册之jwt,以google api为例
cover: /img/development-manual/jwt_title.png
subtitle: 开发手册之jwt,以google api为例
categories: "开发手册"
tags: "开发手册"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-jwt
---

JWT,JSON WEB TOKEN的缩写.其存在的目的是为了在网络应用间传递声明的一种基于JSON格式的标准.



该标准定义在 RFC7519 文件中,可通过下列的连接访问



```
https://tools.ietf.org/html/rfc7519
```



其使用的场景一般存在于分布式站点的单点登录(SSO). 其用途是用来在身份提供者和服务提供者间传递被认证的用户信息.



服务提供者验证JWT的合法性,倘若合法,便会为服务访问者提供服务.



## 1.使用场景



### 01.用户代理与服务器之间



![图片](/1.png)



step 1 : 输入用户名和密码

step 2 : 携带用户名和密码,发起HTTP请求

step 3 : 验证用户名和密码的正确性

step 4 : 验证合法,返回JWT TOKEN

step 5 : 使用服务

step 6 : 携带JWT TOKEB,发起HTTP请求

step 7 : 验证JWT TOKEN的合法性

step 8 : Token合法,返回数据

step 9 : 通过UI呈现给用户



### 02.服务与服务之间



![图片](/2.png)





step 1 : 创建JWT TOKEN,向服务B发起获取访问令牌的请求

step 2 : 验证JWT TOKEN的合法性

step 3 : 返回访问令牌

step 4 : 携带访问令牌,访问服务B的API



## 2.应用举例



用户代理与服务之间比较常见,下面就以服务与服务之间的认证举例,其案例是GOOGLE API中的OATH2.0 SERVICE ACCOUNT 流程.



### 01.在GOOGLE API CONSOLE 创建服务账号.



创建服务账号后,你会拿到如下列的JSON文件



```
{
  "type": "service_account",
  "project_id": "flowing-precept-285304",
  "private_key_id": "7f7a09bf7f9e9ec758409bb5e43e8f218237e65e",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQCoHvXGNLDa6pXt\nI7kv9K3J6HlEv3J42cy87Og4b9nUlUeVZ6AQBnFfwIGY8+4E8dpMs1oYAFTjcFch\nmg2eM28MTCSKS9+7CKSLAHEZS4eFo108ostBbvJPdMDwcQPQ38/VzOtcS8rwI/tv\n/1vrhUed6hOIhKAhPzIyMKOvyQkhzuo1PthHxIxLxg48NfjWqIcIR2D8v4SAAV6s\nm8iXsOIU1fp+r4Y5EPYJcGKt9j/eXtxHnP7sNmpX1/ves79mAIJBgEej/4Jm4xlx\nyDm7wjj+CBTBikrrW5RvSu5JgJu0euJsGEtzd4QMFYGNxqtGpiGDiRR4TWAbbdNz\nrVs4a4hJAgMBAAECggEAFOt9U8OcujD0pQSL966vrW8zH93exbD8bAniv5sTdQN6\nW9oALd5PX0XaGolH9e+OZXrv3Aq2hXKmNPUxep0V1WboKRlV5rUlnHJaoHYoj/WL\nFY+AUU0X89EobQLzIZuoBgewxdRclVM053PUIVN9XOYStisireBqQ5qP08DlVQJx\nnqipOyJFurjVw/IXkpaqhuxju+jWtFEr+JVrhPsqV5/7mCAWNZY1EUgWHxjWjvRc\nateVujbhZranEjDkhkGvTBNklaw78SNUq6VYX2G5B91CzLNeIdXQnJa5PFV2u18P\npArXzPNrz7eGMAGDE3kQozTsVRUp86kli9dsWjhwnQKBgQDdfssyEHg1eNUCvpLx\nVHfBC9DSK+v5nvE70yamUcS6na6dD/CvDHYmWJ7kLNW1zv/hVbKEth07eI2NOM9b\np3Eb5V3key8CDui7yZ/5areosfOBxfpSbcHe2p5VkEV3VEz1kTJc4rsBJSkLJ4Rl\n6mr9nArcwJapC9AORw/85rBoPwKBgQDCT5p8ACCFQKtnEUbg0ADVodZwa0mOa+ms\nEhN4hnFGzRAYxn7agxN19SdGdtz+t+qBS6IyFE0+JDGrvchNi7Lq/2moSDZBE9lp\nFY6eiCmIzrlMIfMEWbCcfG3hnQz2dGb2tXbHTPxmrS8EHeD8N8fdvC+okidQ+v0h\nx8kHP+gtdwKBgC/RwQLFBX7d4HcgN888YkJeT64gZ2jUBNbaplyACM4VXu5v05Gn\nShbLSTqP52/CCgJXIxx9yN/fDghwPGxYQRY5tcSvR53VJC/uvsf1X0Nfb+gTmxCS\nu6lmX4qvhB/YJmlZ+JqPJLqBkFPlKzNpocGxH7M7LQvADiIW+3+pOmq3AoGAcRBM\nvdZ9FcxZb/GXonyl36j51BQ5isuz/lHOTpU8GIx9z0zAx3j5u+tYXSIQ2Y4+v9k4\nmZdCkuQQmvQlNyoQg7j2y9qo5xkbqo/Gmuxz7o0LOQeQFnnx0Dx+24a84jM9LlTM\nto9PVpdzAhw4q8nxXE6CFL5mbjJ9VEih6rv+52UCgYBxvL/CP26pHjc/6Lty1r+5\nhFX6et6awX0KXI9/JUtb1bcv9gKTCtfvieaovKHlkvjpTHYDEw89JKxcf6cpDca2\nmy0cI/8q1aV3EK5yWN5KJwh13+jmA4gAgNdT91RiWE5Mhgp8Y+OaoNWzJ1WDLu31\npNN8Yrbi9RNznlbe1m30Qw==\n-----END PRIVATE KEY-----\n",
  "client_email": "id-741@flowing-precept-285304.iam.gserviceaccount.com",
  "client_id": "104490524108382551606",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/id-741%40flowing-precept-285304.iam.gserviceaccount.com"
}
```

 

#### 02.创建JWT



JWT的格式,诸如下面的格式.



```
{Base64url encoded header}.{Base64url encoded claim set}.{Base64url encoded signature}
```





JWT Header



标头由两个字段组成，这两个字段指示签名算法和令牌的格式. 这两个字段都是必需的，每个字段只有一个值。随着其他算法和格式的引入，此标题将相应地更改



```
{"alg":"RS256","typ":"JWT"}
```



alg 代表签名算法, typ代表令牌的格式



JWT签名算法中，一般有两个选择，一个采用HS256,另外一个就是采用RS256.



签名实际上是一个加密的过程，生成一段标识（也是JWT的一部分）作为接收方验证信息是否被篡改的依据。



RS256 (采用SHA-256 的 RSA 签名) 是一种非对称算法, 它使用公共/私钥对: 标识提供方采用私钥生成签名, JWT 的使用方获取公钥以验证签名。由于公钥 (与私钥相比) 不需要保护, 因此大多数标识提供方使其易于使用方获取和使用 (通常通过一个元数据URL)。

另一方面。

HS256 (带有 SHA-256 的 HMAC 是一种对称算法, 双方之间仅共享一个 密钥。由于使用相同的密钥生成签名和验证签名, 因此必须注意确保密钥不被泄密。

使用RS256更加安全。



而此处的JWT TOKEN采用RSA256



JWT Claim Set



JWT声明集包含有关JWT的信息，包括请求的权限（作用域）、令牌的目标、颁发者、令牌的颁发时间以及令牌的生存期。大多数字段都是必需的。与JWT头一样，JWT声明集是一个JSON对象，用于计算签名。



其格式如下



```
{
  "iss": "761326798069-r5mljlln1rd4lrbhg75efgigp36m78j5@developer.gserviceaccount.com",
  "scope": "https://www.googleapis.com/auth/devstorage.read_only",
  "aud": "https://oauth2.googleapis.com/token",
  "exp": 1328554385,
  "iat": 1328550785
}
```



iss : 服务账号的邮箱,即令牌为谁颁发.

scope : 一个以空格分隔的权限域,即该令牌要访问哪些域

aud : 令牌的颁发者.

exp : 令牌的过期时间.

iat : 令牌的颁发时间





JWT Signature



最后一部分就是JWT的签名,计算签名时必须使用JWT头中的签名算法.googleoauth2.0授权服务器支持的唯一签名算法是使用SHA-256哈希算法的RSA. 这在JWT头的alg字段中表示为RS256.





RSA-SHA256(

{Base64url encoded header}.{Base64url encoded claim set},

私钥,

)

最终,你会得到一个诸如下面格式的token



```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiI3NjEzMjY3OTgwNjktcjVtbGpsbG4xcmQ0bHJiaGc3NWVmZ2lncDM2bTc4ajVAZGV2ZWxvcGVyLmdzZXJ2aWNlYWNjb3VudC5jb20iLCJzY29wZSI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL2F1dGgvcHJlZGljdGlvbiIsImF1ZCI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL29hdXRoMi92NC90b2tlbiIsImV4cCI6MTMyODU1NDM4NSwiaWF0IjoxMzI4NTUwNzg1fQ.UFUt59SUM2_AW4cRU8Y0BYVQsNTo4n7AFsNrqOpYiICDu37vVt-tw38UKzjmUKtcRsLLjrR3gFW3dNDMx_pL9DVjgVHDdYirtrCekUHOYoa1CMR66nxep5q5cBQ4y4u2kIgSvChCTc9pmLLNoIem-ruCecAJYgI9Ks7pTnW1gkOKs0x3YpiLpzplVHAkkHztaXiJdtpBcY1OXyo6jTQCa3Lk2Q3va1dPkh_d--GU2M5flgd8xNBPYw4vxyt0mP59XZlHMpztZt0soSgObf7G3GXArreF_6tpbFsS3z2t5zkEiHuWJXpzcYr5zWTRPDEHsejeBSG8EgpLDce2380ROQ
```



#### 03.发起请求,获得访问令牌



通过下列的API可以获得访问令牌



```
Host : https://oauth2.googleapis.com
Uri : /token
Header :
    Content-Type: application/x-www-form-urlencoded
Paramter :
    grant_type: urn:ietf:params:oauth:grant-type:jwt-bearer
    assertion: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiI3NjEzMjY3OTgwNjktcjVtbGpsbG4xcmQ0bHJiaGc3NWVmZ2lncDM2bTc4ajVAZGV2ZWxvcGVyLmdzZXJ2aWNlYWNjb3VudC5jb20iLCJzY29wZSI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL2F1dGgvcHJlZGljdGlvbiIsImF1ZCI6Imh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL29hdXRoMi92NC90b2tlbiIsImV4cCI6MTMyODU1NDM4NSwiaWF0IjoxMzI4NTUwNzg1fQ.UFUt59SUM2_AW4cRU8Y0BYVQsNTo4n7AFsNrqOpYiICDu37vVt-tw38UKzjmUKtcRsLLjrR3gFW3dNDMx_pL9DVjgVHDdYirtrCekUHOYoa1CMR66nxep5q5cBQ4y4u2kIgSvChCTc9pmLLNoIem-ruCecAJYgI9Ks7pTnW1gkOKs0x3YpiLpzplVHAkkHztaXiJdtpBcY1OXyo6jTQCa3Lk2Q3va1dPkh_d--GU2M5flgd8xNBPYw4vxyt0mP59XZlHMpztZt0soSgObf7G3GXArreF_6tpbFsS3z2t5zkEiHuWJXpzcYr5zWTRPDEHsejeBSG8EgpLDce2380ROQ
```



通过访问上面的请求,便可以得到诸如下面的JSON结构



```
{
  "access_token": "1/8xbJqaOZXSUZbHLl5EOtu1pxz3fmmetKx9W8CV4t79M",
  "scope": "https://www.googleapis.com/auth/prediction"
  "token_type": "Bearer",
  "expires_in": 3600
}
```



其中 access_token 就是访问token 



参考连接



\1. https://developers.google.com/identity/protocols/oauth2/service-account