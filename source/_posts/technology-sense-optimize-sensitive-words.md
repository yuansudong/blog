---
title: 技术场景之优化敏感词屏蔽
cover:  /img/technology-sense/optimize_sensitive_words_title.png
subtitle: 技术场景之优化敏感词屏蔽
categories: "技术场景"
tags: "技术场景"
author: 
nick: 袁苏东
link: https://github.com/yuansudong
---

随着国家对互联网管控的加压,一些涉黄,涉爆,涉政的相关词条不允许在网络上传播.



这一规定的出现,使得一些应用,比如,社交类,游戏类不得不对相关词语进行屏蔽,否则会招到制裁.



## 1.问题场景



为了符合相关法律法规,以及运营的需求.我们将敏感词的配置放在了数据库中,以方便运营在后台实时地增减相关词条.



每当敏感词库更新之后,后台需要通知相关API服务器敏感词有了变化.相关API服务器收到通知后,会通过HTTP协议重新从资源服务器上拉取敏感词,并对该服的敏感词进行更新.



最开始,敏感词的屏蔽,同事采用的是正则表达式替换.但是,随着敏感词条的增加,一个接口的处理时间达到了4秒之多.



4秒,已经超出了用户能忍受的极限.优化的脚步不得不提前.



## 2.解决方案



通过调研以及测试,最终选择了TrieTree作为优化方案.



记忆中TrieTree最早出现在算法导论二叉树那章的练习题中,当时没怎么在意,在调研后,才发现,原来它还有如此的作用.



Trie，又称单词查找树或键树，是一种树形结构，是一种哈希树的变种。



典型应用是用于统计和排序大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。



它的优点是：最大限度地减少无谓的字符串比较，查询效率比哈希表高.



它的特点是: 根节点不包含字符，除根节点外每一个节点都只包含一个字符。

从根节点到某一节点，路径上经过的字符连接起来，为该节点对应的字符串。

每个节点的所有子节点包含的字符都不相同。





了解到TrieTree之后,还需要了解golang中rune类型.用过GO的人都知道,GO中的字符串是由rune类型构成的.



在在源代码的builtin.go文件中,有下面的几行代码定义了rune的实际类型.



```go
// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```



从上面的代码中,可以看到,rune的实际类型是int32.



对于数据, 在计算机底层存储都是0和1.字符串也不例外.字符串在转换成byte之后,其实都是一个数值.

英文字符是ASCII码来存储的,占用一个字节.而中国以及日本的常用文字在4千以上,一个字节肯定是存储不了的.

而此时,这些存储不下的文字,会通过unicode存储.也就是两个字节.

但是,你会发现.当按照上面的定义去言明字符串的字节时,是错误的. 比如,下面的这个代码



```go
package main

import "log"

func main() {

  strA := "abcdef" // 目测为6
  strB := "ab哈希哈"  // 目测为8
  fmt.Println("strA", len(strA), "strB", len(strB))
}
```



输出结果为:



```
strA 6 strB 11
```



惊讶吧? 竟然是11.



至于出现11的原因,这和该字符的编码有关系.经过测试,得出下面的结论.



0 ~ 2的8次方            1个字节

2的8次方 ~ 2的16次方    2个字节

2的16次方  ~ 2的24次方  3个字节

2的24次方 ~ 2的32次方   4个字节



"哈"的编码为21704,占用3个字节. "希"的编码为24076,占用3个字节.而a,b皆占用一个字节.所以,最后求出的长度为11



## 3.TireTree实现



### 01.结点定义



```go
// _Node Trie树上的一个节点.
type _Node struct {
  isRootNode bool // 用于表示是否是bool结点
  isPathEnd  bool // 用于表示该结点是否是末尾结点
  Character  rune // 当前结点所存储的字符
  Children   map[rune]*_Node // 当前字符后面跟随的有那些字符
}
```



### 02.TireTree定义



```go
// _TrieTree 短语组成的Trie树.
type _TrieTree struct {
  Root *_Node // 描述一个根结点
}
```



### 03.增加



```go
func (tree *_TrieTree) add(word string) {
  var current = tree.Root
  var runes = []rune(word)
  for position := 0; position < len(runes); position++ {
    r := runes[position]
    if next, ok := current.Children[r]; ok {
      current = next
    } else {
      newNode := _NewNode(r)
      current.Children[r] = newNode
      current = newNode
    }
    if position == len(runes)-1 {
      current.isPathEnd = true
    }
  }
}
```



### 04.删除



```go

// del 删除单个
func (tree *_TrieTree) del(word string) {
  var current = tree.Root
  var runes = []rune(word)
  for position := 0; position < len(runes); position++ {
    r := runes[position]
    if next, ok := current.Children[r]; !ok {
      return
    } else {
      current = next
    }
    if position == len(runes)-1 {
      current.SoftDel()
    }
  }
}
```



### 05.替换



```go
// Replace 词语替换
func (tree *_TrieTree) Replace(text string, character rune) string {
  var (
    parent  = tree.Root
    current *_Node
    runes   = []rune(text)
    length  = len(runes)
    left    = 0
    found   bool
  )

  for position := 0; position < len(runes); position++ {
    current, found = parent.Children[runes[position]]

    if !found || (!current.IsPathEnd() && position == length-1) {
      parent = tree.Root
      position = left
      left++
      continue
    }
    if current.IsPathEnd() && left <= position {
      for i := left; i <= position; i++ {
        runes[i] = character
      }
    }
    parent = current
  }
  return string(runes)
}
```



### 06.查找



```go

func (tree *_TrieTree) Find(text string) (bool, string) {
  const (
    Empty = ""
  )
  var (
    parent  = tree.Root
    current *_Node
    runes   = []rune(text)
    length  = len(runes)
    left    = 0
    found   bool
  )
  for position := 0; position < len(runes); position++ {
    current, found = parent.Children[runes[position]]
    if !found || (!current.IsPathEnd() && position == length-1) {
      parent = tree.Root
      position = left
      left++
      continue
    }
    if current.IsPathEnd() && left <= position {
      return false, string(runes[left : position+1])
    }
    parent = current
  }
  return true, Empty
}
```





更新之后,接口从4s多,转变为毫秒级