---
title: 技术场景之计算大文件hash值
cover:  /img/technology-sense/sum_big_file_hash_value_title.png
subtitle: 技术场景之计算大文件hash值
categories: "技术场景"
tags: "技术场景"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: images
---

在应用开发中,通常需要存储用户的图像,以及各式各样的文件.因此,一个应用通常伴随着文件上传服务.



由于网络,或者人为的因素可能会终止文件的上传. 因此,上传的文件,都需要对文件求一个HASH值.



该HASH值,可以用于避免重复上传.以及检验文件的完整性.



有一个十分大的文件,比如64G.此时需要对该文件求HASH值,以事实HASH值,与参数中的HASH值验证,用于判定此时磁盘上的中间文件是否是一个完整的文件.



在一些应用中,是不允许上传如此大的文件.倘若,用户上传了如此大的文件,可以从参数中获取上传的文件大小.根据文件大小是否超过指定的阈值.



如果超过,则拒绝该文件的上传.如果没有,则接受该文件的上传.



但是,在一些极端的应用中,用户就是要上传如此大的文件.这使得程序必须去解决大文件带来的内存开销的问题.



## 1.问题场景



在文件上传的服务中,第一个版本,前辈采用了一次性读取文件数据到内存中.然后,将内存中的文件数据写到磁盘上,并求出一个MD5的HASH值.



服务器通过该HASH值,与客户端所带的参数中的HASH值进行比较.倘若相等,则证明该文件是完整的.



倘若不相等,则证明文件是不完整的.检测到不完整之后,向客户端返回相应的错误码.



此种方式,对于小型文件,不会出问题.对于大型文件,肯定会出现内存不够用,从而触发LINUX的OOM.



## 2.解决办法



通过一番调研之后,发现除了一次性将数据交给HASH函数的方法之外,还有一种分块的方式.



GO中的代码如下



```go

import "crypto/md5"
import "log"
import "fmt"

func main() {
  md5h :=  md5.New()
  body := make([]byte,1024)
  md5h.Write(body)
  log.Println(fmt.Sprintf("%x",md5h.Sum(nil)))
}

```



方法找到之后,就需要对其进行测试.通过GO中的基准测试,得出下列的数据.



### 01.对10G文件进行压测



```go
Running tool: C:\Go\bin\go.exe test -benchmem -run=^$ -bench . blog.ysd.com\compare_md5

2021/02/09 13:29:05 f76fd62ea13fb9ba52b0d90be7f60f35
goos: windows
goarch: amd64
pkg: blog.ysd.com/compare_md5
BenchmarkAllMd5-16              1  148678694500 ns/op  11394925680 B/op       872 allocs/op
2021/02/09 13:31:05 f76fd62ea13fb9ba52b0d90be7f60f35
BenchmarkBufMd5-16              1  118891937900 ns/op      7176 B/op        14 allocs/op
2021/02/09 13:32:22 f76fd62ea13fb9ba52b0d90be7f60f35
BenchmarkCopyMd5-16             1  77299399700 ns/op     36080 B/op        17 allocs/op
PASS
ok    blog.ysd.com/compare_md5  347.011s
```



从上面可以看出,AllMd5花费时间,花费内存最多,其次是BufMd5,最后是CopyMd5.



下面是相关代码



```go

package compare_md5

import (
  "bufio"
  "crypto/md5"
  "fmt"
  "io"
  "io/ioutil"
  "log"
  "os"
)

const _FilePath = "E://1//2.txt"

// ReadAllMd5 读取全部的md5
func ReadAllMd5() {
  data, err := ioutil.ReadFile(_FilePath)
  if err != nil {
    log.Fatalln(err.Error())
  }
  fmt.Sprintf("%x", md5.Sum(data))
}

// BufMd5 具有缓冲区的md5
func BufMd5() {
  fileHandler, err := os.Open(_FilePath)
  if err != nil {
    log.Fatalln(err.Error())
  }
  defer fileHandler.Close()
  reader := bufio.NewReader(fileHandler)
  md5Handler := md5.New()
  _, err = io.Copy(md5Handler, reader)
  if err != nil {
    log.Fatalln(err.Error())
  }
  fmt.Sprintf("%x", md5Handler.Sum(nil))
}

// CopyMd5 使用md5拷贝的方式
func CopyMd5() {
  fileHandler, err := os.Open(_FilePath)
  if err != nil {
    log.Fatalln(err.Error())
  }
  defer fileHandler.Close()
  md5Handler := md5.New()
  _, err = io.Copy(md5Handler, fileHandler)
  if err != nil {
    log.Fatalln(err.Error())
  }
  fmt.Sprintf("%x", md5Handler.Sum(nil))
}
```



看了上面的代码是不是很奇怪.为什么一次性读取文件到内存,其执行速度为什么会那么慢.



按理来说,不应该是越来越快么?



在一番猜测下,应该是本机内存用尽,使用了SWAP导致.



### 02.对一个9KB的文件进行基准测试



```
pkg: blog.ysd.com/compare_md5
BenchmarkAllMd5-16           2070      513634 ns/op     10337 B/op         8 allocs/op
BenchmarkBufMd5-16           2833      498403 ns/op      5052 B/op         9 allocs/op
BenchmarkCopyMd5-16          2695      428254 ns/op     33647 B/op         8 allocs/op
PASS
ok    blog.ysd.com/compare_md5  6.117s
```



从上面的测试中,可以看出,BufMd5所用内存最少,其次是AllMd5,最后是CopyMd5.



经过查看源代码.



io.Copy() 该函数,会默认分配一个32K的缓冲区



```
func Copy(dst Writer, src Reader) (written int64, err error) {
  return copyBuffer(dst, src, nil)
}
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
  if wt, ok := src.(WriterTo); ok {
    return wt.WriteTo(dst)
  }
  if rt, ok := dst.(ReaderFrom); ok {
    return rt.ReadFrom(src)
  }
  if buf == nil {
    size := 32 * 1024 //  此处会分配一个32K的缓冲区
    if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
      if l.N < 1 {
        size = 1
      } else {
        size = int(l.N)
      }
    }
    buf = make([]byte, size)
  }
  for {
    nr, er := src.Read(buf)
    if nr > 0 {
      nw, ew := dst.Write(buf[0:nr])
      if nw > 0 {
        written += int64(nw)
      }
      if ew != nil {
        err = ew
        break
      }
      if nr != nw {
        err = ErrShortWrite
        break
      }
    }
    if er != nil {
      if er != EOF {
        err = er
      }
      break
    }
  }
  return written, err
}
```



而 bufio.NewReader(fileHandler),会默认分配一个4K的缓冲区.



在一番比对下,最终采用用bufio.NewReader用于从socket中读取文件数据,在写入的过程中,计算md5值.



只不过,使用了bufio.NewReaderSize(fileHandler,8K).即分配了一个8K的缓冲区.

