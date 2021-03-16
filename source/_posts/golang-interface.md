---

title: 聊聊golang中的interface
cover: /img/golang/interface_title.png
subtitle: 聊聊golang中的interface
categories: "GO语言"
tags: "开发手册"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-jwt
---

GO是静态编译语言,自2009发布至今,已有11年之久.其主要应用于云原生,微服务,web开发,运维工具等场景.



除此之外,GO也进入了数据科学(GO+),嵌入式等领域.



在我的理解中,GO既是面向对象语言也不是面向对象语言.因为,GO有面向对象编程的类型和方法的概念,但是却没有继承这一说.



在GO中,接口定义了一套方法的集合,任何实现这些方法的struct都可以被认为实现了这个接口.



而之所以需要接口(interface),是因为在日常开发中,接口可以屏蔽实现细节,保护结构体不被无意修改.



除此之外,接口(interface)还可以达到先设计后编写的目的.比如服务器中的会话管理.而这个会话可以是tcp也可以是websocket,通过定义接口ISession,让tcp会话结构体(struct)或者websocket会话结构体(struct),达到类型一致,业务功能上无差别的目的



## 1.标准定义



在GO中,标准库对于接口方法的定义遵循以职能为单元,组合单元形成功能的原则



比如,标准包中的ReadWriter接口,该部分的代码在io包下的io.go的文件中.



```go
// Reader is the interface that wraps the basic Read method.
//
// Read reads up to len(p) bytes into p. It returns the number of bytes
// read (0 <= n <= len(p)) and any error encountered. Even if Read
// returns n < len(p), it may use all of p as scratch space during the call.
// If some data is available but not len(p) bytes, Read conventionally
// returns what is available instead of waiting for more.
//
// When Read encounters an error or end-of-file condition after
// successfully reading n > 0 bytes, it returns the number of
// bytes read. It may return the (non-nil) error from the same call
// or return the error (and n == 0) from a subsequent call.
// An instance of this general case is that a Reader returning
// a non-zero number of bytes at the end of the input stream may
// return either err == EOF or err == nil. The next Read should
// return 0, EOF.
//
// Callers should always process the n > 0 bytes returned before
// considering the error err. Doing so correctly handles I/O errors
// that happen after reading some bytes and also both of the
// allowed EOF behaviors.
//
// Implementations of Read are discouraged from returning a
// zero byte count with a nil error, except when len(p) == 0.
// Callers should treat a return of 0 and nil as indicating that
// nothing happened; in particular it does not indicate EOF.
//
// Implementations must not retain p.
type Reader interface {
  Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
//
// Write writes len(p) bytes from p to the underlying data stream.
// It returns the number of bytes written from p (0 <= n <= len(p))
// and any error encountered that caused the write to stop early.
// Write must return a non-nil error if it returns n < len(p).
// Write must not modify the slice data, even temporarily.
//
// Implementations must not retain p.
type Writer interface {
  Write(p []byte) (n int, err error)
}

// ReadWriter is the interface that groups the basic Read and Write methods.
type ReadWriter interface {
  Reader
  Writer
}
```



在上面的定义中,我们可以看到ReadWriter中嵌套了两个接口,他们分别是Reader和Writer.



Reader定义了Read方法,倘若函数参数值是io.Reader,那么传入的结构体(struct)只要实现了Read方法就可以被当做参数传入.



Writer定义了Write方法,倘若函数参数值是io.Writer,那么传入的结构体(struct)只要实现了Write方法就可以被当做参数传入.



而ReadWriter嵌套了Reader和Writer,倘若让io.ReadWriter作为参数传递或者返回值返回,则需要结构体实现Reader中的Read方法和Writer中的Write方法.



在开源社区,接口方法的定义其遵循的原则是



```ABAP
接口嵌入接口，保持深度在0或1为最佳。
接口中直接定义的方法数量10个之内最佳。
```