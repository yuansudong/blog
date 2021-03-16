---
title: 技术场景之优化邀请码
cover:  /img/technology-sense/optimize_invation_code_title.png
subtitle: 技术场景之优化邀请码
categories: "技术场景"
tags: "技术场景"
author: 
  nick: 袁苏东
  link: https://github.com/yuansudong
---

邀请码是识别邀请人的一种方式,常常被用作推广. 而邀请码最便捷的方式就是使用邀请人的UID.



但是,根据接口开发规范,数据库字段不能暴露在接口中,特别是UID.因此,邀请码一般是由一个[A-Z],[a-z],[0-9]的字符串组成.



对于一些工程,它们是通过生成一批预备邀请码在数据库中.



在用户申请邀请码的时候,将邀请码与申请人的UID绑定在一起,并且给这个绑定一个过期时间.



当未知用户注册时,通过填写邀请码的方式,便可以将邀请人和新用户绑定在一起,以及对邀请人发放利益.



## 1.问题场景



(1) 邀请码过多,增加数据库查询负担.



(2) 减少邀请码,随着用户量的增长.用户会频繁收到邀请码不够用的信息.



## 2.解决办法 





在一番权衡之后,采用了进制转换.即将UID的十进制,转换为其他进制.



在计算机系统中,有进制一说.比如,二进制,八进制,十进制,十六进制.其核心便是逢二进一,逢八进一,逢十进一,逢十六进一.但是在核心的最前面,却是基数.



二进制的基数为 [ 0 , 1 ].

八进制的基数为 [ 0 , 1 , 2 , 3 , 4 , 5 , 6 , 7 ].

十进制的基数为 [ 0 , 1 , 2 , 3 , 4 , 5 , 6 , 7 , 8 , 9 ].

十六进制的基数为 [ 0 , 1 , 2 , 3 , 4 , 5 , 6 , 7 , 8 , 9 , A , B , C , D , E , F ].



对于邀请码而言,短小是核心,所以邀请码的最大长度通常被产品限制为六位.



由此可见,十进制是不能向二进制和八进制转换的.因为,这会使得邀请码变得非常的长.



至于十六进制,在一定程度上可以容纳更多的十进制.但是,它会很快地突破六位的限制,且被大部分程序员所熟识.因为不能作为邀请码.首先会喷开发者没见识.其次,会暴露邀请人的核心信息.



因此,于邀请码而言, 需要自己设计一个新的进制出来. 上面提到 , 邀请码由[A-Z],[a-z],[0-9]组成.



但是,由于 1和 l,0和O对于人而言不太容易区分,且常常区分错误. 所以,要将l,1,O,0在基数中除掉.



```
{
    'A', 'B', 'C', '6', 'D', 'E', 'F', '2', 'G', 'H', 'J', '3', 'K', 'L', 'M', '4', 'N', 'P', '5', 'Q', 'R', 'S', '7', 'T', '8', 'U', 'V', 'W', '9', 'X', 'Y', 'Z',
    'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'j', 'k', 'm', 'n', 'p', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z',
}
```



因为,基数中有54个基数,因此将其称之为54进制.

对于6位数的邀请码而言,54进制可以容纳 54 x 54 x 54 x 54 x 54 x 54 个十进制进制.

即 24794911296个十进制,这个数字是两百亿之多, 比全球的人还多.



因此,以此种方式生成邀请码,将邀请码与用户绑定在一起,基本上可以满足所有大型应用.从而不必担心,查询效率的问题.更不要担心数据库



所以,邀请码的生成方式,决定将用户的UID转换成以54为基数的字符串.



下面是GO的实现代码.主要目的是起一个抛砖引玉的目的



## 3.实施步骤



### 01.前提准备

```
// base 用于存储进制转换函数
var base []byte

// basemap 用于根据值获取索引,以此来提高索引的效率
var basemap map[byte]int

var binary int64

var defaultNum int

// 用于初始化base中的编码, 即进制转换的函数
func init() {
  base = []byte{
    'A', 'B', 'C', '6', 'D', 'E', 'F', '2', 'G', 'H', 'J', '3', 'K', 'L', 'M', '4', 'N', 'P', '5', 'Q', 'R', 'S', '7', 'T', '8', 'U', 'V', 'W', '9', 'X', 'Y', 'Z',
    'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'j', 'k', 'm', 'n', 'p', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z',
  }
  basemap = make(map[byte]int)
  for index := 0; index < len(base); index++ {
    basemap[base[index]] = index
  }
  defaultNum = 6            //  默认给6位
  binary = int64(len(base)) //  用于求出这是多少进制的
}
```



### 02.将用户的UID编码为邀请码



```
// InvationEncode 对数字进行编码,获得54进制类似的邀请码
func InvationEncode(num int64) string {
  store := make([]byte, 0, defaultNum)
  l := list.New()
  encode(num, l, 1)
  // 查看位数填充,序列集合上抽像的0值
  for index := l.Len(); index < defaultNum; index++ {
    store = append(store, base[0])
  }
  for curr := l.Front(); curr != nil; curr = curr.Next() {
    store = append(store, curr.Value.(byte))
  }
  return string(store)
}

// encode 是Encode 的辅助函数
func encode(encodeNum int64, l *list.List, index int) {
  // merchant 用于求商
  merchant := encodeNum / binary
  // remainder 用于求余
  remainder := encodeNum % binary
  // 余数是上一位的的取值,商数是当前位的取值
  l.PushFront(base[int(remainder)])
  // 检查商是否是需要进1, 如果需要进1, 需要进入下一轮
  if merchant >= binary {
    encode(merchant, l, index+1)
  } else {
    if merchant != 0 {
      l.PushFront(base[int(merchant)])
    }
  }
}
```



### 03.将邀请码解码为用户的UID



```
// InvationDecode 用于将相应的字符串邀请码转换成用户的UID,int64
func InvationDecode(str string) (int64, bool) {
  if str == "" {
    return 0, false
  }
  store := []byte(str)
  length := len(store)
  value := int64(0)
  for index := length; index > 0; index-- {
    pos := index - 1
    num, isExists := basemap[store[pos]]
    if !isExists {
      return value, false
    }
    value += int64(num) * getDividend(length-index, binary)
  }
  return value, true
}
// getDividend 用于获取被除数
func getDividend(c int, bin int64) int64 {
  sum := int64(1)
  for index := 0; index < c; index++ {
    sum *= bin
  }
  return sum
}
```



### 04.测试



十进制 					 五十四进制

10010						AAA6TR

20020					  AAAFsj

12345678910	    VtvHv9