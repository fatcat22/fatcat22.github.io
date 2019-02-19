---
layout: post
title:  "go语法：将方法转换成普通函数"
categories: golang
tags: 原创 golang method_expression
excerpt: 你是否知道，在go语言里，方法（method）也可以变成普通函数被调用呢？
author: fatcat22
---

* content
{:toc}





在看以太坊的代码时，看到有这样一条语句：
```go
  runtime.SetFinalizer(c, (*cache).finalizer)
```

其中`cache`是一个结构体，而`c`是一个`*cache`类型的变量。`*cache`有一个名为`finalizer`的方法，定义如下：
```go
func (c *cache) finalizer() {
  .......
}
```

 这里让我感到疑惑的是`(*cache).finalizer`这个表达式。先不管`runtime.SetFinalizer`对参数有什么要求，仅仅是这个表达式，我也没搞明白它是什么意思，甚至它怎么会是一个合法的表达式呢？

 于是我尝试一顿搜索，终于找到了[解释这个问题的官方文档](https://golang.org/ref/spec#Method_expressions)。

 从这个文档中可以看出来，这种表达式有一个专门的名称，叫做"method expression"，中文可以翻译为“方法表达式”。简单来说，在go语言中，如果你为某个类型T定义了一个方法v，那么使用“T.v”这种方式，就可以生成一个和方法v相同的函数，但这个函数增加了一个参数，即第一个参数新增为一个T类型的参数。

 文字描述比较绕，我们直接看例子吧。拿文章开头提到的表达式来说，由于`finalizer`是`*cache`类开的方法，且它的定义是这样的：
```go
func (c *cache) finalizer() {
  .......
}
```

那么`(*cache).finalizer`就生成了一个函数，这个函数应该是这样定义的：
```go
f := func (c *cache) {
  .......
}
```

除此之外，它们的的函数体内容完全一致。

其实在[描述"方法"的文档里](https://golang.org/ref/spec#Method_declarations)，第一句话就是：
> A method is a function with a receiver

即“方法是一种有接收者的函数”。以这种设计思路来看，“方法表达式”也就不奇怪了。


 回头来看`runtime.SetFinalizer`的文档是这样描述它的参数的：
> func SetFinalizer(obj interface{}, finalizer interface{})  
> The argument finalizer must be a function that takes a single argument to which obj's type can be assigned, and can have arbitrary ignored return values.

也就是说它的第二个参数是一个函数，这个函数必须满足两个条件：
1. 仅有一个参数，且`runtime.SetFinalizer`的第一个参数`obj`可以正常赋值给这个参数
2. 返回值可以被忽略

根据上面对“方法表达式”的描述，`(*cache).finalizer`正好可以生成一个符合`runtime.SetFinalizer`第二个参数要求的函数。
