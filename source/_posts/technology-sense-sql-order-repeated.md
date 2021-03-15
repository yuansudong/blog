---
title: 技术场景之解决排序引起的数据重复
cover:  /img/technology-sense/data_repeated_title.png
subtitle: 技术场景之解决排序引起的数据重复
categories: "技术场景"
tags: "技术场景"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: images
---

列表,在实际开发中是一个非常常见的功能.比如,广场,附近的人都是基于列表所展示的.但是,对于非个人数据的列表.比如广场,会时不时的出现重复卡片的问题.



经过排查,发现出现重复数据的原因,是因为服务器接口返回的数据重复,从而导致客户端展示出有重复数据的问题,也导致了用户体验极差的结果.



下面,就对于出现的原因进行分析



在列表中,通常需要按照一个维度进行排序.下面以广场这个常见的功能模块进行说明.



广场,其筛选条件包含着距离最近,点赞最多,最新发布,及其性别等一些列的筛选条件.



其中,距离最近,点赞最多,最新发布为互斥条件,三者只能选其一,且三种都要通过排序来达到效果.



距离最近,以用户发布动态的经纬度为准,计算两个点之间的距离,升序排序.其公式如下.



```
Distance = R*Arccos(sin(MLatA)*sin(MLatB)*cos(MLonA-MLonB) + cos(MLatA)*cos(MLatB))*Pi/180
```



点赞最多,以动态表中,字段为点赞的个数,进行降序排序.



最新发布,以动态表中,创建时间为维度,进行降序排序.



对于这些排序,在静态数据下,不会出现重复的ITEM.但是,倘若在浏览期间,有其他用户发布了最新的动态,此时对于在浏览期间的用户,便会看到有重复的ITEM出现.



记得第一次遇见这类情形时,大脑中懵然一片.在模拟用户流程之后,得出了造成这一现象的原因是排序造成的.



比如,现有一个有序列表[5,4,3,2,1],且列表中的数据不能被移除.



用户A想浏览这个列表.其方式是通过每次从列表中获取两个ITEM,现在已经获取了2次,偏移量为4.第一次获取[5,4],第二次获取[3,2].



就在此时,用户B向列表中插入了一个新的ITEM,其值为6.此时,这个有序列表就变成了[6,5,4,3,2,1]



用户A第三次获取列表中的ITEM,通过前两次获取,用户A处于偏移量为4的位置.根据规则,第三次用户A拿到的数据为[2,1].即,用户A三次获得的数据为[5,4],[3,2],[2,1].



下面我们就解决办法做出讨论



#### 01.基于REDIS缓存



通过redis中的list结构,为每个用户临时维护一个浏览的list.从而解决数据重复的问题.  



优点:各类客户端以服务器为准,可以高度统一.



缺点:基于缓存做,那么在一定时间间隔内,用户无法看到最新状态



结论:产品不允许,且服务器内存消耗,不合适



#### 02.服务器基于数据的临界值,进行筛选



在SQL优化中,有一段经验,对于数据筛选排序,须尽量避免使用OFFSET,采用临界值的方式缩小数据集,进行TOPN查询.



但是,需得注意的是,对于此类用法有一个前提,其排序字段需要有序性和唯一性.否则,会出现漏数据或者数据重复的问题.下面以最新发布为例.



在最新发布中,按照创建时间降序排序. 而在APP设计中,产品并没有说明在1秒内,只允许一个用户发布动态.



这也就决定了在1秒内可能会有N个用户发布动态,也就造成了创建时间的可重复性.



下面是一个SQL的举例



```
select {需要的数据} from user_dynamic where dynamic.create_at < {{客户端最后一条数据的时间戳}}   order by dynamic.create_at desc limit 50
```



对于上面的SQL语句,假设客户端给的时间戳为100,且在100这个时间戳内,有123条动态.如果用小于进行筛选,最大可漏数据为53.



```
select {需要的数据} from user_dynamic where dynamic.create_at <= {{客户端最后一条数据的时间戳}}   order by dynamic.create_at desc limit 50
```



对于上面的sql语句,假设客户端给的时间戳为100,且在100这个时间戳内,有123条动态.如果用小于等于进行筛选,最大重复数据为无限.



之所以无限,是因为我取50条,最后一条数据的时间戳还是100.这会造成无限循环.



#### 03.客户端去重



客户端在获取数据时,在本地维护一个set或者hashmap的数据集.其唯一key为动态的唯一ID.



每次获取数据后,在展示之前,先通过set或者dictionary进行查询,发现有重复数据时,客户端不予以展示,并且会在广场的滚动视图的顶端对用户进行微小的提示.



比如,有新动态啦! 有人距离你更进一步! 等一些列俏皮词汇,并且给与用户能回滚到最顶端,重新查看的功能.