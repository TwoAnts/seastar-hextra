---
title: Effective Go 小记
description: Effective Go阅读笔记
slug: note-effective-go
date: 2022-07-16 15:30:39+0800
image:
categories:
    - learn
tags:
    - golang
    - language
toc: true
---

# Effective Go

* 指针和值的方法有所不同。针对值的方法，可以作用于值和指针；针对指针的方法，只能作用于指针。  
  主要的不同是 __值的方法无法修改值本身，而指针可以__ 。  
  例外：当对值调用原本作用于指针的方法时，如果值可以取地址，则编译器会帮忙转换。  

  ```go
  type Size int
  func (s *Size) Add(a Size) {}

  var s Size
  s.Add(1) // 等价于 (&s).Add(1)，此处编译器帮忙进行了转换
  ```

<!--more-->

* 类型转换 可以帮忙程序调用正确的方法

* 接口转换(interface conversion) vs 类型断言(type assertion)
  ```go
  type Stringer interface {
      String() string
  }

  var value interface{} // Value provided by caller.
  switch str := value.(type) {
  case string:
      return str
  case Stringer:
      return str.String()
  }
  ```

  类型断言可以帮忙把value中的指定类型成员拿出来使用，如果没有，则会抛runtime err或将ok置false
  ```go
  if str, ok := value.(string); ok {  // 尝试类型断言
      return str
  } else if str, ok := value.(Stringer); ok { //
      return str.String()
  }
  ``` 

* 除了指针和接口，方法可以针对任何类型定义，甚至可以 __为函数类型定义方法__
  ```go
  type Handler interface {
      ServeHTTP(ResponseWriter, *Request)
  }

  // The HandlerFunc type is an adapter to allow the use of
  // ordinary functions as HTTP handlers.  If f is a function
  // with the appropriate signature, HandlerFunc(f) is a
  // Handler object that calls f.
  type HandlerFunc func(ResponseWriter, *Request)

  // ServeHTTP calls f(w, req).
  func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
      f(w, req)
  }

  // Argument server.
  func ArgServer(w http.ResponseWriter, req *http.Request) {
      fmt.Fprintln(w, os.Args)
  }

  http.Handle("/args", http.HandlerFunc(ArgServer))

  ```

* _可以用来将无用的变量赋值给它。大多是为了避免编译错误。
  * 有些变量暂时用不到
    ```go
    package main

    import (
        "fmt"
        "io"
        "log"
        "os"
    )

    var _ = fmt.Printf // For debugging; delete when done.
    var _ io.Reader    // For debugging; delete when done.

    func main() {
        fd, err := os.Open("test.go")
        if err != nil {
            log.Fatal(err)
        }
        // TODO: use fd.
        _ = fd
    }
    ```
  * 有些import包，只是想要它的副作用，而不会真正使用它
    ```go
    import _ "net/http/pprof"
    ```
  * 有时候想在编译期检查类型转换是否ok
    ```go
    var _ json.Marshaler = (*RawMessage)(nil)
    ```
  * 判断value是否实现了该接口
    ```go
    if _, ok := val.(json.Marshaler); ok {
      fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
    }
    ```

* 内嵌type或*type可以让类具有type相同的方法，当然也可以自行定义struct的同名方法起到覆盖的作用。
  多个内嵌类型方法/成员同名冲突的解决规则:
  * 越表层的方法/成员 掩盖 越底层(深度)的方法/成员
  * 当外部从未访问过该方法/成员，同名冲突可以直接忽略，否则会报错

## Concurrency

> Do not communicate by sharing memory; instead, share memory by communicating.
> 不要通过共享内存来通信；通过通信来共享内存。

goroutine: 在相同地址空间与其他goroutine一起并发执行的函数。(与线程、协程、进程的概念区分开)

goroutine __轻量__ ，开销基本就是栈空间的分配。栈空间初始很小，所以分配开销小，当需要时通过分配堆存储来增长。

goroutine 与 OS线程是 __多对多映射__ ，当发生阻塞(等待IO)时，其他goroutine继续运行。其中隐藏掉了很多线程创建和管理的复杂操作。

go中的匿名函数是 __闭包__ ，函数中涉及到的变量会在函数执行过程中一直存活。

go闭包错误示例：
```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
// req被循环的多轮共享，所以多个goroutine处理了相同的req
```
更正为:
```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```
或者
```go
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // Create new instance of req for the goroutine.
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
// req := req 可以遮挡循环本身的req，这样每个goroutine会获得不同的req
```

如果想起到控制并发度的效果，也可以直接拉起固定并发的若干goroutine，然后所有goroutine共享同一channel
```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // Wait to be told to exit.
}
```

* channel也可以是channel的元素, 比如chan chan int😂

* Parallelization

runtime.NumCPU()可以告知CPU数目
runtime.GOMAXPROCS(0) 返回默认的工作线程数，缺省值为NumCPU()，可以手动设置

## Panic Recover

Recover可以用来将模块内部的panic转化为普通的error。如果此处是其他panic，代码中会在
type assert时触发runtime error，导致程序停止。
```go
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()
    return regexp.doParse(str), nil
}
```






