---

title: 聊聊golang中,与JSON相关的使用技巧
cover: /img/golang/json_title.png
subtitle: 聊聊golang中,与JSON相关的使用技巧
categories: "GO语言"
tags: "GO语言"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-jwt
---

JSON（JavaScript Object Notation，JavaScript对象表示法）是一种由道格拉斯·克罗克福特构想和设计、轻量级的资料交换语言，该语言以易于让人阅读的文字为基础，用来传输由属性值或者序列性的值组成的数据对象。尽管JSON是JavaScript的一个子集，但JSON是独立于语言的文本格式，并且采用了类似于C语言家族的一些习惯。



其应用领域广泛,常被用作web开发,API接口开发,配置文件,NoSQL数据库



## 1.基本类型



JSON的基本数据类型有,数值,字符串,布尔值,数组,对象,null



数值: 十进制,不能有前导0,可为负数,可为浮点数,亦可以有e或者E表示指数部分



字符串: 以双引号""括起来的零个或多个Unicode码位,支持反斜杠开始的转义字符



布尔值: 表示为true或者false



数组: 有序的零个或者多个值.每个值可以为任意类型.其形式由[]括起来,元素只见用逗号隔开.



对象: 由{}包括,若干个key-val组成.其K-V之间用:分隔.



null类型: 其值为null



## 2.使用技巧



在GO开发中,JSON的定义常常与结构体相关,通过在结构体字段中的tag,达到控制JSON的目的



### 01.通过使用omitempty,当字段为默认值时,不序列化字段



在API接口开发过程中,有些时候响应的JSON数据中,有些字段是默认值,比如0,"",null.



为了避免流量消耗,对于这些默认值,可以在序列化时,将其抛弃.例子如下



```go
package main

import (
  "encoding/json"
  "log"
)

// Old 原始结构
type Old struct {
  Name    string `json:"name"`
  Age     int    `json:"age"`
  Address string `json:"address"`
  Sex     *int   `json:"sex"`
}

// New 改变结构
type New struct {
  Name    string `json:"name,omitempty"`
  Age     int    `json:"age,omitempty"`
  Address string `json:"address"`
  Sex     *int   `json:"sex,omitempty"`
}

func main() {
  // now := time.Now()
  mOld := new(Old)
  // mOld.Time = now
  bOldData, mOldErr := json.Marshal(mOld)
  if mOldErr != nil {
    log.Fatalln(mOldErr.Error())
  }
  log.Println("old ==> ", string(bOldData))
  mNew := new(New)
  // mNew.Time = now
  bNewData, mNewErr := json.Marshal(mNew)
  if mNewErr != nil {
    log.Fatalln(mOldErr.Error())
  }
  log.Println("new ==> ", string(bNewData))
}
```



输出结果如下



```log
old ==>  {"name":"","age":0,"address":"","sex":null}
new ==>  {"address":""}
```



从结果中可以看到,tag中有omitempty修饰的字段,当其值是空值时,不会被序列化



### 02.可用"-"忽略某个字段,让其不加入序列化



在实际开发中,结构体中的有些字段需要在接口代码中使用,但是不想将其呈现在接口返回数据中.



此时,我们可以在tag中使用"-",将其忽略掉.倘若你的JSON对象的key是"-",那么需要使用"-,",达到不被屏蔽的效果.



示例代码如下



```json
package main

import (
  "encoding/json"
  "fmt"
  "log"
)

// Old 原始结构
type Old struct {
  Name    string `json:"name"`
  Age     int    `json:"age"`
  Address string `json:"address"`
  Sex     *int   `json:"sex"`
  Ignore  string `json:"ignore"`
  Dec     string `json:"-"`
}

// New 改变结构
type New struct {
  Name    string `json:"name,omitempty"`
  Age     int    `json:"age,omitempty"`
  Address string `json:"address"`
  Sex     *int   `json:"sex,omitempty"`
  Ignore  string `json:"-"`
  Dec     string `json:"-,"`
}

func main() {
  // now := time.Now()
  mOld := new(Old)
  // mOld.Time = now
  bOldData, mOldErr := json.Marshal(mOld)
  if mOldErr != nil {
    log.Fatalln(mOldErr.Error())
  }
  fmt.Println("old ==> ", string(bOldData))
  mNew := new(New)
  // mNew.Time = now
  bNewData, mNewErr := json.Marshal(mNew)
  if mNewErr != nil {
    log.Fatalln(mOldErr.Error())
  }
  fmt.Println("new ==> ", string(bNewData))
}
```



输出结果如下



```
old ==>{"name":"","age":0,"address":"","sex":null,"ignore":""}
new ==>{"address":"","-":""}
```



### 03.通过实现json.Marshaler接口和json.Unmarshaler接口控制序列化



在实际开发中,GO中的时间操作都是以time.Time为单位.但是,json在序列化时将time.Time序列化成了字符串,其格式为time.RFC3339Nano.反序列化时需要传递同等格式字符串才能被反序列化成时间对象.



因为时区问题的存在,所以有关于时间,服务器以及上传参数中都是以UNIX时间戳的方式来表达时间.



其实,以RFC3339Nano的形式返回时间,客户端也是能够转换时间到本地时间的.但是,怎奈习惯一经形成,想让别人改变.



一般会回绝你四个字--"好麻烦啊"



所以,以时间戳作为参数和返回数据中的时间,已经成为了一种常态.此时,我们可以通过实现json.Marshaler接口和json.Unmarshaler,来控制实例化.



[GO之理解接口在开发中的应用](https://www.yuansudong.top/2021/golang-interface/index.html)



而相关方法的生成,可通过编写protobuf的插件来达到避免每个结构体都要手写一次的体力活



相关代码如下



```json
package main

import (
  "encoding/json"
  "fmt"
  "log"
  "time"
)

// Old 原始结构
type Old struct {
  Name    string    `json:"name"`
  Age     int       `json:"age"`
  Address string    `json:"address"`
  Sex     *int      `json:"sex"`
  Ignore  string    `json:"ignore"`
  Dec     string    `json:"-"`
  Time    time.Time `json:"time"`
}

// New 该收
type New struct {
  Name      string    `json:"name,omitempty"`
  Age       int       `json:"age,omitempty"`
  Address   string    `json:"address"`
  Sex       *int      `json:"sex,omitempty"`
  Ignore    string    `json:"-"`
  Dec       string    `json:"-,"`
  Time      time.Time `json:"-"`
  Timestamp int64     `json:"timestamp"`
}

// UnmarshalJSON 用于反序列化JSON
func (n *New) UnmarshalJSON(data []byte) error {
  type Alise New
  n1 := new(Alise)
  if err := json.Unmarshal(data, n1); err != nil {
    log.Println(err.Error())
    return err
  }
  n1.Time = time.Unix(n1.Timestamp, 0)
  *n = *(*New)(n1)
  return nil
}

// MarshalJSON 实现json.Marshaler
func (n *New) MarshalJSON() ([]byte, error) {
  type Alise New
  n.Timestamp = n.Time.Unix()
  n1 := (*Alise)(n)
  return json.Marshal(n1)
}

func main() {
  now := time.Now()
  mOld := new(Old)
  mOld.Time = now
  bOldData, mOldErr := json.Marshal(mOld)
  if mOldErr != nil {
    log.Fatalln(mOldErr.Error())
  }
  fmt.Println("old ==> ", string(bOldData))
  mNew := new(New)
  mNew.Time = now
  bNewData, mNewErr := json.Marshal(mNew)
  if mNewErr != nil {
    log.Fatalln(mOldErr.Error())
  }
  fmt.Println("new ==> ", string(bNewData))
  mUnNew := new(New)
  json.Unmarshal([]byte(`{"address":"erdads","-":"","timestamp":1615315849}`), mUnNew)
  fmt.Printf("%+v \n", mUnNew)
}
```



输出结果如下:



```
old ==>  {"name":"","age":0,"address":"","sex":null,"ignore":"","time":"2021-03-10T02:58:32.8308486+08:00"}
new ==>  {"address":"","-":"","timestamp":1615316312}
&{Name: Age:0 Address:erdads Sex:<nil> Ignore: Dec: Time:2021-03-10 02:50:49 +0800 CST Timestamp:1615315849}
```



### 04.为什么在实现json.Marshaler和json.Unmarshaler时,需要起别名



原因是,倘若还是使用New结构体,将New结构体放入json.Marshal()和json.Unmarshal() 这会导致无限递归.



最终程序会出现下列错误,崩溃掉



```
runtime: goroutine stack exceeds 1000000000-byte limit
runtime: sp=0xc020161388 stack=[0xc020160000, 0xc040160000]
```



通过定义 type Alise New 的形式该New重新定义了一个类型,该类型定义时,Alise会保留New中的所有字段,但是不会保留New中的方法,从而规避了无限递归.