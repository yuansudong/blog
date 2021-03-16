---

title: 聊聊golang中,与channel相关的使用技巧
cover: /img/golang/channel_title.png
subtitle: 聊聊golang中,与channel相关的使用技巧
categories: "GO语言"
tags: "GO语言"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-jwt
---

channel是Go中的一个核心类型，你可以把它看成一个管道，通过它并发核心单元就可以发送或者接收数据进行通讯(communication)。



channel具有四个6个特性,它们分别是有缓冲的channel,无缓冲的channel,只读channel和只写channel,打开的channel关闭的channel.



## 1.关于channel



### 01.channel的打开和关闭



channel的打开通过make(chan int),channel的关闭通过内置函数close



```go

package main

func main() {
 ch :=  make(chan int)
 close(ch)
}
```



### 02.有缓冲的channel和无缓冲的channel



在写入的过程中,有缓冲的channel除非缓冲区已满,否则执行流程不会阻塞.无缓冲的channel则会被阻塞,直到无缓冲的channel中的元素被读取为止.



在读取的过程中,对于有缓冲的channel,只要channel中有元素就会读取.对于无缓冲的channel,需要等待其他协程写入



```go

package main

import (
  "context"
  "fmt"
  "log"
  "sync"
  "time"
)
func main() {
  BlockChannel()
  NonBlockChannel()
}

// BlockChannel 阻塞channel
func BlockChannel() {
  var wg sync.WaitGroup
  cCh := make(chan int, 1)

  ctx, cancel := context.WithCancel(context.Background())
  wg.Add(2)
  // 写入
  go func(wCtx context.Context, wCh chan<- int) {
    defer wg.Done()
    i := 1
    for {
      select {
      case <-wCtx.Done():
        return
      case wCh <- i:
      default:
        log.Println("Write处于阻塞状态")
      }
      i++
    }
  }(ctx, cCh)
  go func(rCtx context.Context, rCh <-chan int) {
    defer wg.Done()
    for {
      select {
      case <-rCtx.Done():
        return
      case ele := <-rCh:
        log.Println("接收新值:", ele)
      default:
        log.Println("Read处于阻塞状态")
      }
    }
  }(ctx, cCh)
  time.Sleep(5 * time.Second)
  cancel()
  wg.Wait()
  close(cCh)
  log.Println("结束!")

}

// NonBlockChannel 非阻塞的channel
func NonBlockChannel() {
  cChannel := make(chan int, 1)
  cChannel <- 1
  log.Println(<-cChannel)
  close(cChannel)
}
```



### 03.只读的channel和只写的channel



channel具有只读和只写的特性,只读的channel只能从channel内接收元素,不能向channel里写入元素.



只写的channel只能向管道内写入元素,不能从管道内读取元素



```go

package main

import (
  "context"
  "fmt"
  "log"
  "sync"
  "time"
)

func main() {
  RWChannel()
}
func RWChannel() {
  var wg sync.WaitGroup
  cChannel := make(chan int, 1)
  ctx, cancel := context.WithCancel(context.Background())
  wg.Add(2)
  go func() {
    defer wg.Done()
    ReadHandler(ctx, cChannel)
  }()
  go func() {
    defer wg.Done()
    WriteHandler(ctx, cChannel)
  }()
  time.Sleep(5 * time.Second)
  cancel()
  wg.Wait()
  close(cChannel)
  log.Println("结束!")

}

func ReadHandler(ctx context.Context, rCh chan int) {
  for {
    select {
    case <-ctx.Done():
      return
    case ele := <-rCh:
      log.Println(ele)
    }
  }
}

func WriteHandler(ctx context.Context, wCh chan int) {
  i := 0
  for {
    select {
    case <-ctx.Done():
      return
    default:
      i++
      wCh <- i
    }
  }
}
```



### 04. 只读和只写的channel



对于只读channel,可以读取channel中的元素,但是,不能写入和关闭



对于只写channel,可以向channel中写入元素,但是,不能读取和关闭



```go
type Model struct {
 // 只读
  _ReadChannel <-chan int 
  // 
  _WriteChannel chan<- int
}
```



下面是一个只读和只写的channel demo



```go

package main

import (
  "context"
  "fmt"
  "log"
  "sync"
  "time"
)

func main() {
  RWChannel()
}
func RWChannel() {
  var wg sync.WaitGroup
  cChannel := make(chan int, 1)
  ctx, cancel := context.WithCancel(context.Background())
  wg.Add(2)
  go func() {
    defer wg.Done()
    ReadHandler(ctx, cChannel)
  }()
  go func() {
    defer wg.Done()
    WriteHandler(ctx, cChannel)
  }()
  time.Sleep(5 * time.Second)
  cancel()
  wg.Wait()
  close(cChannel)
  log.Println("结束!")

}

func ReadHandler(ctx context.Context, rCh <-chan int) {
  //// rCh 只能读,不能写,不能被关闭,否则编译不过
  for {
    select {
    case <-ctx.Done():
      return
    case ele := <-rCh:
      log.Println(ele)
    }
  }
}

func WriteHandler(ctx context.Context, wCh chan<- int) {
  // wCh 只能写,不能读,不能被关闭,否则编译不过
  i := 0
  for {
    select {
    case <-ctx.Done():
      return
    default:
      i++
      wCh <- i
    }
  }
}
```



## 2.关于select



谈到了channel,肯定离不开select



Go中的select是Golang在语言层面提供的I/O多路复用的机制，其专门用来检测多个channel是否准备完毕：可读或可写。



其大致结构如下



```go
select {
case r1 := <- ch1:
  log.Println(r1)
case r2 :=  <- ch2:
  log.Println(r2)
default:
  log.Println("default")
}
```



看到select的结构,是不是觉得和switch很像. 其实他和switch有着诸多的区别.



下面,我们来探索下,select的神奇之处



01.select中,对于case的选择是随机的.



为了验证case是随机的,下了下面的例子.



```go

package main
import "fmt"
func main() {
  ch1 := make(chan int, 1)
  ch2 := make(chan int, 1)
  go func() {
    ch1 <- 1
  }()
  go func() {
    ch2 <- 2
  }()
  select {
  case v1 := <-ch1:
    fmt.Println("v1:", v1)
  case v2 := <-ch2:
    fmt.Println("v2:", v2)
  default:
    fmt.Println("default")
  }
}
```



对于上面的例子,输出可能如下



第一种: v1:1

第二种: v2:2

第三种: default



### 02.select中,如果没有default语句，则会阻塞等待任一case



```go
package main
import "fmt"
func main() {
  ch1 := make(chan int, 1)
  ch2 := make(chan int, 1)
  select {
  case v1 := <-ch1:
    fmt.Println("v1:", v1)
  case v2 := <-ch2:
    fmt.Println("v2:", v2)
  }
}
```



输出结果如下



```
fatal error: all goroutines are asleep - deadlock!
```



### 03.select中,对于接受的channel,一定要判断管道是否关闭.



要判断管道是否关闭,因为管道关闭后,返回的是一个当前元素的默认值.



```go
package main

import "fmt"

func main() {
  ch1 := make(chan *int, 1)
  ch2 := make(chan *int, 1)
  close(ch1)
  select {
  case v1,ok := <-ch1:
    fmt.Println("v1:", v1,ok)
  case v2 := <-ch2:
    fmt.Println("v2:", v2)
  }
}
```



输出结果如下



```
v1: <nil> false
```



### 04.select中,如果什么语句都没有,则会阻塞住当前的协程.最终报死锁的错误



如果select语句中什么都没有,协程就会被阻塞住.



```go
package main
func main() {
  select {}
}
```



输出结果



```
fatal error: all goroutines are asleep - deadlock!
```



## 3.使用技巧



### 01.channel关闭时,读取该channel的协程都会收到通知



基于该特性,我们可以用一个channel,通知所有的协程退出.此种情况,一般发生在进程收到进程的退出信号,做进程退出前的清理工作时.



样例代码:



```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  cExit := make(chan struct{})
  wg.Add(3)
  go func() {
    defer wg.Done()
    select {
    case <-cExit:
      fmt.Println("g1 退出")
      return
    }
  }()
  go func() {
    defer wg.Done()
    select {
    case <-cExit:
      fmt.Println("g2 退出")
      return
    }
  }()
  go func() {
    defer wg.Done()
    select {
    case <-cExit:
      fmt.Println("g3 退出")
      return
    }
  }()
  close(cExit)
  wg.Wait()
}
```



输出结果:



```
g2 退出
g3 退出
g1 退出
```



### 02.基于channel的可读可写特性,可以保护参数式channel不被胡乱读取或者写入,从而导致程序写入死锁.



样例代码:



```go
func WriteExample(ctx context.Context, ch chan<- int ) {}
func ReadExample(ctx context.Context, ch <-chan int){}
```



如此写法,可以确保channel在WriteExample只能写,不能读不能关闭.在ReadExample中只能读,不能写,不能关闭



### 03.基于向已经关闭的channel中写入会发生异常的特性,可以简化代码

### 04.基于timer可做协程的驻留时间,以及异步任务的超时操作



说到这里,肯定会有人反驳.因为,在写程序的时候,一直遵循着谁开启谁关闭,谁申请谁释放的原则.



这个原则是正确的.但是对于一些场景,你发现背道而驰会意想不到的简洁.



下面就以实现驻留式协程池为例.



驻留式协程池要实现的目标是,对协程的复用,避免频繁开启协程所带来的花销.



```go
package main

import (
  "context"
  "log"
  "math/rand"
  "sync"
  "sync/atomic"
  "time"
)

// Worker 用于描述一个工人
type Worker struct {
  _WID    uint64
  _TaskCh chan int
}

// NewWorker 新建立一个工人
func NewWorker(iWID uint64) *Worker {
  return &Worker{
    _WID:    iWID,
    _TaskCh: make(chan int),
  }
}

// Do 用于工人执行任务
func (w *Worker) Do(ctx context.Context) {
  tIdleDuration := 500 * time.Millisecond
  for {
    select {
    case <-ctx.Done():
      log.Println("进程退出")
      goto end
    case <-time.After(tIdleDuration):
      log.Println("Worker驻留时间已到准备退出!", w._WID)
      goto end
    case task := <-w._TaskCh:
      time.Sleep(time.Second) // 模拟任务处理时长
      log.Println("Worker:", w._WID, " 收到任务", task)
    }
  }
end:
  close(w._TaskCh)
  return
}

// Leader 用于管理工人
type Leader struct {
  sync.WaitGroup
  ctx    context.Context
  idx    uint64
  cancel context.CancelFunc
  works  map[uint64]*Worker
}

// NewLeader 实例化一个leader
func NewLeader() *Leader {
  mCtx, mCancel := context.WithCancel(context.Background())
  return &Leader{
    ctx:    mCtx,
    idx:    0,
    cancel: mCancel,
    works:  make(map[uint64]*Worker),
  }
}

// ClearWorker 用于清理工人
func (l *Leader) ClearWorker(w *Worker) {
  delete(l.works, w._WID)
}

// DispatchTask 用于派发任务
func (l *Leader) DispatchTask(iTask int) {
  bSuccess := false
  for _, work := range l.works {
    if l.NotifyWorker(work, iTask) {
      bSuccess = true
      break
    }
  }
  if !bSuccess {
    l.WorkerStartup(iTask)
  }
}

// NotifyWorker 用于通知
func (l *Leader) NotifyWorker(w *Worker, iTask int) (status bool) {
  defer func() {
    if r := recover(); r != nil {
      // 进入这里代表工人已经退出了
      l.ClearWorker(w)
      log.Println("清理了工人:", w._WID)
      return
    }
  }()
  select {
  case w._TaskCh <- iTask:
    status = true
    return
  default:
    return
  }
}

// WorkerStartup 启动一个工人
func (l *Leader) WorkerStartup(iTask int) {
  w := NewWorker(atomic.AddUint64(&l.idx, 1))
  go func() {
    l.Add(1)
    defer l.Done()
    w.Do(l.ctx)
  }()
  w._TaskCh <- iTask
  l.works[w._WID] = w
}

// Exit 用于退出操作
func (l *Leader) Exit() {
  // 向所有协程通知退出信号
  l.cancel()
  // 等待所有协程退出
  l.Wait()
  log.Println("开启协程的最大编号是:", l.idx)
}

func main() {
  mLeader := NewLeader()
  for i := 0; i < 100; i++ {
    mLeader.DispatchTask(i)
    // 模拟任务的不确定性
    time.Sleep(time.Duration(rand.Intn(10)) * time.Second)
  }
  time.Sleep(15 * time.Second)
  mLeader.Exit()
}
```



输出样例:



```log
2021/03/13 09:24:38 Worker: 1  收到任务 0
2021/03/13 09:24:38 Worker驻留时间已到准备退出! 1
2021/03/13 09:24:39 Worker: 2  收到任务 1
2021/03/13 09:24:39 Worker驻留时间已到准备退出! 2
2021/03/13 09:24:45 清理了工人: 1
2021/03/13 09:24:45 清理了工人: 2
2021/03/13 09:24:46 Worker: 3  收到任务 2
2021/03/13 09:24:46 Worker驻留时间已到准备退出! 3
2021/03/13 09:24:52 清理了工人: 3
2021/03/13 09:24:53 Worker: 4  收到任务 3
2021/03/13 09:24:53 Worker驻留时间已到准备退出! 4
2021/03/13 09:25:01 清理了工人: 4
2021/03/13 09:25:02 Worker: 5  收到任务 4
2021/03/13 09:25:02 Worker驻留时间已到准备退出! 5
2021/03/13 09:25:03 Worker: 6  收到任务 5
2021/03/13 09:25:03 Worker驻留时间已到准备退出! 6
2021/03/13 09:25:10 清理了工人: 5
2021/03/13 09:25:10 清理了工人: 6
2021/03/13 09:25:11 Worker: 7  收到任务 6
2021/03/13 09:25:11 Worker驻留时间已到准备退出! 7
2021/03/13 09:25:15 清理了工人: 7
2021/03/13 09:25:16 Worker: 9  收到任务 8
2021/03/13 09:25:16 Worker: 8  收到任务 7
2021/03/13 09:25:16 Worker驻留时间已到准备退出! 9
2021/03/13 09:25:16 Worker驻留时间已到准备退出! 8
2021/03/13 09:25:21 清理了工人: 8
2021/03/13 09:25:21 清理了工人: 9
2021/03/13 09:25:22 Worker: 10  收到任务 9
2021/03/13 09:25:22 Worker: 11  收到任务 10
2021/03/13 09:25:22 Worker驻留时间已到准备退出! 11
2021/03/13 09:25:22 Worker驻留时间已到准备退出! 10
```