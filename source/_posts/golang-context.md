---

title: 聊聊golang中,与context相关的使用技巧
cover: /img/golang/context_title.png
subtitle: 聊聊golang中,与context相关的使用技巧
categories: "GO语言"
tags: "GO语言"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-jwt
---

context,译为上下文,自go1.7引入,按照官方所述,它是一个请求的全局上下文,携带了截止时间,手动取消等信号,并且包含一个并发安全的map,可以用于传递数据.context被定义在官方包中,其结构如下



```go
type Context interface {
  //Deadline返回代表此上下文完成工作的时间
  //应该取消。没有指定截止日期时，Deadline返回ok==false,否则返回true
  //设置。连续调用Deadline返回相同的结果。
  Deadline() (deadline time.Time, ok bool)
  // 通过select case <-Done() 可以主动退出
  Done() <-chan struct{}
  // 判断context是否是正常退出
  Err() error
  //  可通过key取得与key对应的val
  Value(key interface{}) interface{}
}
```



## 1.使用技巧



### 01.基于context.WithValue向下文传值.



```go
package main

import (
  "context"
  "log"
)
type _ContextWithValue struct {
  key string
  val string
}
func _NewContextWithValue() *_ContextWithValue {
  return &_ContextWithValue{
    key: "hello",
    val: "world",
  }
}
func (cwv *_ContextWithValue) Do() {
  cwv.Func1(context.Background())
}
func (cwv *_ContextWithValue) Func1(ctx context.Context) *_ContextWithValue {

  cwv.Func2(context.WithValue(ctx, cwv.key, cwv.val))
  return cwv
}

// Func2 用于获得func1设置的值
func (cwv *_ContextWithValue) Func2(ctx context.Context) *_ContextWithValue {
  log.Println(ctx.Value(cwv.key))
  return cwv
}

```



通过调用Do函数,会输出world.



需要注意的是,此种方式需要注意的是,子context可以通过ctx.Value(key)得到父context设置的val.



但是,父context不能通过ctx.Value(key)得到子context的值



### 02.基于context.WithTimeout可以对异步任务设置超时.



在日常开发中,这种情况情况挺多,比如通过RPC,去后台服务获取数据.而在RPC调用中,不可能无休止的等待下去.



所以,一般需要基于context.WithTimeout做超时处理



样例代码如下



```go
package main

import (
  "context"
  "time"
)

type _Request struct {
}

type _Response struct {
}

//
type _ContextWithTimeout struct {
}

func (cwt *_ContextWithTimeout) Do() {
  ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()
  cwt.handle(ctx, new(_Request))
  time.Sleep(2 * time.Second)
}

func (cwt *_ContextWithTimeout) handle(ctx context.Context, req *_Request) (*_Response, error) {
  cRspChan := make(chan *_Response)
  go func(cReplyChan chan<- *_Response) {
    time.Sleep(2 * time.Second) // 模拟rpc请求
    cReplyChan <- new(_Response)
  }(cRspChan)
  select {
  case <-ctx.Done():
    return nil, ctx.Err()
  case rsp := <-cRspChan:
    return rsp, nil
  }
}

```



### 03.基于context.WithCancel让进程优雅的退出.



在服务器开发中,基本一个进程需要开启很多协程用于异步完成任务.但是golang的协程没有主动打断的办法,只有等待协程执行完毕才表示协程退出.



我们可以通过调用context.WithCancel返回的cancel函数,让派生自该context的所有子context触发 <- ctx.Done() 执行,从而退出协程.



样例代码如下



```
func _ContextWithCancel() {
  ctx, cancel := context.WithCancel(context.Background())
  go func() {
    aCtx, aCancel := context.WithCancel(ctx)
    aCancel()
    select {
    case <-aCtx.Done():
      log.Println("A协程退出")
    }
  }()
  go func() {
    select {
    case <-ctx.Done():
      log.Println("B协程")
    }
  }()
  time.Sleep(5 * time.Second)
  cancel()
  time.Sleep(5 * time.Second)
  log.Println(ctx.Err())
}
```



需要注意的是,父context的cancel被调用.子context的Done()会收到信号通知,子context的cancel被调用.父context的Done()不会收到信号通知.



因此,在进程监听到linux系统发出的关闭信号时,调用cancel函数,通知所有的协程退出.



然后,基于sync.Waitgroup中的Wait()等待所有协程退出,是非常合适的.