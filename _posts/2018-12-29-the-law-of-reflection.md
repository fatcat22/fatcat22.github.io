---
layout: post
title:  "golang:反射的规则"
categories: golang
tags: golang 翻译 reflection
author: fatcat22
---

* content
{:toc}





本文是对Go Blog上[一篇文章](https://blog.golang.org/laws-of-reflection)的翻译和自己的简单总结。

# 翻译

### 引言
在计算机中，反射（reflection）指的是一个程序检查自身结构的能力，尤其是通过各个类型进行检查。它是元编程的一种方式，很容易引起人们的困惑。

在这篇文章中我们尝试通过解释Go语言中反射是如何工作的来消除你的困惑。每种语言的反射模型都是不一样的（很多语言也根本不支持反射），但这篇文章中我们只谈论Go中的反射。所以后面当我们再次提到“反射”这个词时，我们的意思是“Go中的反射”。

### Types and interface
Go是一门强类型的语言，让我们先来重温一下Go中关于类型的知识吧。

Go是静态类型的语言，每一个变量都有自己的、一直不会改变的类型。也就是说变量的类型在编译时就已知并且确定下来了，比如int, float32, \*MyType等类型。如果我们声明

```go
type MyInt int

var i int
var j MyInt
```

那么i的类型是int而j的类型是MyInt。变量i和j拥有不同的类型。虽然它们底层的类型是相同的，但它们不能在没有类型转换的情况下直接赋值给彼此。

在所有类型中，interface是一种非常重要的类型，它代表了一组确定的方法（methods）。一个interface类型的变量可以存放任意一个具体类型（非interface类型）的值，只要这个具体的类型实现了这个interface的所有方法。一个大家都熟知的interface类型的例子就是io包中的io.Reader和io.Writer：

```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任何类型实现了Read(或Write)方法，我们就说这个类型实现了io.Reader（或io.Writer）接口。从本篇文章的目的的角度来说，这意味着一个io.Reader类型的变量可以接受任意一个类型的值，只要这个类型实现了Read方法:

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

需要重点说明的一点是，无论r持有什么具体类型的值，它自己的类型总是io.Reader：要记得Go是一种静态类型的语言，而r的静态类型是io.Reader。

关于interface类型的极其重要的一个例子是空的interface：

```go
interface{}
```

一个空的interface代表了一组空的方法。它可以被任意值赋值，因为任何值都有0个或多个方法。

有人说Go的inteface类型是动态类型，但这是一种误导。它们是静态类型：一个interface类型的变量总是保持它的类型不变，即使在运行时存储在interface变量中的值和类型会变，但interface变量本身一直是它声明时的类型。

我们需要明确的把这些东西弄明白，因为反射和interface是高度相关的。


### interface代表什么

Russ Con已经写过一篇关于Go中的interface的[博文](https://research.swtch.com/2009/12/go-data-structures-interfaces.html)。我们无需在这里再重复一遍，只是对重点作一个简单的总结。

一个interface类型的变量存储了两个信息：值，和这个值的类型信息。具体来说，值就是赋值时的具体数据项，这个数据项肯定实现了interface的所有方法；类型信息就是数据项的完整的类型的信息。比如，这段代码之后：

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

r中包含的信息对(value, type)，为(tty, \*os.File)。（这只是个示意性的写法）。注意\*os.File类型其实还实现了除Read之外的方法。即使interface变量只能访问Read方法，但变量内部其实是保存了类型的所有信息。这就是你为什么可以这样做：

```go
var w io.Writer
w = r.(io.Writer)
```

上面赋值语句中的表达式叫做类型断言（type assertion）；这个表达式断定r内部的数据也实现了io.Writer，所以我们把它赋值给w。赋值以后，w就包含了和r相同的信息：（tty, \*os.File），inteface的静态类型决定了这个类型的变量可以调用哪些方法，即使内部的的值可能有更多的方法。

进一步地，我们可以这样做：

```go
var empty interface{}
empty = w
```

空的interface类型的变量empty仍然会像前面的例子一样，持有数据对（tty, \*os.File）。这就很方便了：一个空的interface可以持有任意的值和值的所有完整信息。

（这里不需要类型断言是因为我们明确的知道w满足空interface类型。在将一个Reader变量赋值给Writer变量的例子里，我们需要显式的使用类型断言是因为Writer的方法集合并非Reader的方法集合的子集）

这里有一个非常重要的细节就是，在interface变量内部的数据对总是（value, 具体类型信息）而不是（value, interface类型信息）。interface变量不会持有interface变量。

现在我们已经准备好了解反射了。

### 反射的第一条规则
1. 反射可以将interface值转变成反射对象(reflection object)

从根本上来说，反射只是一种检查一个interface变量内部的值和类型信息的机制。一开始我们首先需要知道的是reflect包的两个类型：reflect.Type和reflect.Value，这两个类型提供了访问interface变量内部数据的入口；另外还需要知道两个函数：reflect.TypeOf和reflect.ValueOf，这俩函数使用一个interface变量生成reflect.Type值和reflect.Value值。（从reflect.Value中也可以轻易的得到reflect.Type，但目前还是让我们保持Value和Type这俩概念独立吧）

我们先从TypeOf开始：

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```

这段代码输出：
> type: float64

你可以会疑惑上面的例子中interface在哪呢，因为看上去好像传给TypeOf的参数是float64类型的变量x，而不是一个interface类型的变量。但确实有interface变量。查看[文档](https://golang.org/pkg/reflect/#TypeOf)可以看到，reflect.TypeOf的函数声明中包含一个空的interface：

```go
// TypeOf returns the reflection Type of the value in the interface{}.
func TypeOf(i interface{}) Type
```

当我们调用reflect.TypeOf(x)时，x先保存到一个空的interface类型的变量中，然后把这个变量作为参数传入。reflect.TypeOf解析这个空的interface变量并恢复它的类型信息。

显然，reflect.ValueOf函数用来恢复参数的值（从这里开始，我们尽量多看示例代码少说话，因此文字描述上可能并不会很严谨）：

```go
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x).String())
```

输出：
> value: \<float64 Value>

（这里显式调用String方法的原因是fmt包会深入解析一个reflect.Value并打印出内部的具体值，String方法不会）

reflect.Type和reflect.Value都有非常多的方法让我们可能检查和操纵它们自身。一个重要的例子是reflect.Value有一个叫Type的方法会返回它的reflect.Type值。另一个是无论是Type还是Value都有一个Kind()方法，这个方法返回一个常量来表示调用者持有的值的类型：Uint, Float64, Slice等等。另外Value还提供了像Int和Float这种类型的方法，可以让我们获取其内部存储的值：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

输出：
> type: float64  
> kind is float64: true  
> value: 3.4  

Value也提供了SetInt和SetFloat这类方法，但想要使用它们我们等理解settability这个概念，我们会在后面的第三条规则中讲解这个主题。

反射库里有几条特性值得我们拿出来单独说说。首先，为了保持API的简单，Value的"getter"和"setter"方法直接使用了持有的值的最大的类型：比如对于有符号整数直接使用int64类型。也就是说，Value的Int方法返回的是int64且SetInt使用int64作为参数。你可能需要根据情况转换成实际的类型：

```go
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())                                       // v.Uint returns a uint64.
```

第二个特性是一个reflect.Value的Kind值只是说明了一个值的底层类型，而不是这个值本身的类型。如果一个reflect.Value保存了一个用户自定义的整数类型，比如：

```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
```

v.Kind()仍然是reflect.Int，即使x的类型是MyInt，而非int。换句话说，Kind无法区分int和MyInt。不过reflect.Type可以。


### 反射的第二条规则
2. 反射可以将成反射对象(reflection object)转变interface

像物理反射一样，Go中的反射也有逆操作。

给定一个reflect.Value我们可以使用Interface方法将其恢复成一个interface值。实际上这个方法重新将类型和值类似打包成一个interface值并返回：

```go
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}
```

因此我们可以这样操作：

```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

来打印v所代表的float64类型的值。

我们甚至可以做得更好。fmt.Println、fmt.Printf等函数的参数类型都是空的interface，这些参数会在fmt包的内部被解析成它们原本的类型，正如我们刚才做的那样。因此它所需要的参数正好是Interface方法的返回值：

```go
fmt.Println(v.Interface())
```

(为什么不是fmt.Println(v)呢？因为v是一个reflect.Value类型的值，而我们需要的是它所代表的数据)因为这个值是float64类型，我们甚至可以用一个浮点数格式化来输出：

```go
fmt.Printf("value is %7.1e\n", v.Interface())
```

这个例子中会l输出：
> 3.4e+00

再说一遍，没有必要使用类型断言将v.Interface()的返回值转变成float64。空的interface值内部已经拥有了实际值的类型信息，Printf会使用这些信息将正确的值恢复回来。

简单来说，Interface方法是ValueOf的逆操作，除了它的返回值类型是Interface{}。

重申一下：反射可以将interface值转换成反射对象，也可以将其转换回来。


### 反射的第三条规则
3. 只能修改可修改的(settable)反射对象的值

第三条规则是最微妙和令人困惑的，但如果我们从基本原理出发，就很容易理解了。

下面是一段错误的代码，但很值得我们学习一下：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) //Error: will panic
```

如果你运行这段代码，程序将以下面的错误信息退出：
> panic: reflect.Value.SetFloat using unaddressable value

问题不是获取不到7.1的地址，而是v是不可修改的。可修改（settability）是reflect.Value的一个特性，但实际运行时并不是所有reflect.Value变量都有这个特性。

reflect.Value的CanSet方法可以告诉调用者它是否可以修改。在上面的例子中：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

将输出：
> settability of v: false

使用一个不可修改的反射对象调用Set方法会引发错误。但什么是“可修改”呢？

可修改有点像可取地址（adressability），但更严格。它是反射对象可以修改实际的存储空间的一种特性，这些存储空间是在创建这个反射对象时获取到的。可修改与否是由反射对象是否能获得原始的所代表的对象来决定的。当我们写下这样的代码时：

```go
var x float64 = 3.4
v := relfect.ValueOf(x)
```

我们传入reflect.ValueOf的是x的拷贝，所以作为relfect.ValueOf的参数创建的interface值是x的一个拷贝，不是x本身。因此，如果语句 `v.SetFloat(7.2)` 被允许成功执行，它将不会更新x的值，即使v看上去是用x创建的。如果真的成功执行它将会更新在它内部的x的拷贝对象，但x本身不会发生任何改变，但这样的行为非常容易让人困惑并且也没什么用处，所以它是非法的，“可修改”就是用来避免这种问题的一个特性。

可能你会觉得这很奇怪，但其实不是的。这只不过是一种常见情况的不同形式罢了。考虑一下将x传给一个函数:

```go
f(x)
```

我们不会期望f能修改x的值因为我们传入的是x的拷贝，而不是x本身。如果我们想让f可以直接修改x的值我们必须传入x的地址（即一个指向x的指针）：

```go
f(&x)
```

这种情况很直接且常见，反射也是类似的机制。如果我们想用反射修改x，我们需要给反射库里的函数一个指向想修改的值的指针。

我们这就来试试。首先我们像之前一样初始化x，然后创建一个指向它的反射对象p：

```go
var x float64 = 3.4
p := reflect.ValueOf(&x) //Note: take the address of x
fmt.Println("type of p:", p.Type())
fmt.Println("setability of p:", p.CanSet())
```

输出为：
> type of P: \*float64  
> setability of p: false

p仍然不可修改，但我们不是想修改p，而是想修改*p。为了获取p指向的对象，我们需要调用Value对象的Elem方法，这个方法间接的通过指针将结果保存到我们命名的变量v中：

```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

现在v是一个可修改的对象了，正如示例的输出所示：
> settability of v: true

并且因为它代表了x，我们终于可以调用v.SetFloat来修改x的值了：

```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

代码的输出正是我们期望的：
> 7.1  
> 7.1

虽然反射比较难理解但它仍然严格遵守着这门语言的所有规则，尽管能过reflect.Value和reflect.Type可以伪装将要发生的事。时亥要记住，reflect.Value需要一个值的地址才能修改这个值。

##### Struts
在我们上面的例子中v不是指针本身，它只是从一个指针派生出来的。从这种情况衍生出来的一个常见的方式是使用反射修改一个结构体的字段。只要我们拥有这个结构体的地址，我们就可以修改它的字段。

下面是一段解析一个类型为结构体的变量t的简单代码。我们使用了t的地址值创建reflect.Value，因为我们后面要修改它。然后我们将它的类型赋值给typeOfT变量，然后使用了几个通过名字就能明确知道其含义的方法（具体细节请查看[package reflect 文档](https://golang.org/pkg/reflect/)）遍历所有字段。注意我们是通过类型信息获取字段的名字，但字段本身是一个reflect.Value对象：

```go
type T struct {
  A int
  B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
  f := s.Field(i)
  fmt.Printf("%d: %s %s = %v\n", i, typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```

这代码代码的输出为:
> 0: A int = 23  
> 1: B string = skidoo

这里有一个要点需要注意：T的字段的名字是大写的（导出的），因为在结构体中只有导出的字段是可修改的。

因为s是一个可修改的反射对象，我们可以修改结构体的字段的值：

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("sunset Strip")
fmt.Println("t is now", t)
```

输出为:
> t is now {77, Sunset Strip}

如果我们修改代码让s通过t创建而不是&t，那么调用SetInt和SetString就会失败，因为t的各字段是不可修改的。

### 总结
下面是这篇文章中提到的反射的规则：
- 反射可以将interface值转变成反射对象
- 反射可以将反射对象转变成interface值
- 只能修改可修改的(settable)反射对象的值

一旦你理解了这些规则，Go语言的反射就非常易用了，即使仍然有微妙的地方。这是一个非常强大的工具，你应该非常小心的使用甚至避免使用，除非非常需要。

还有很多反射相关的知识我们没有提到：channel的发送和接收，申请内存，使用slice和map，调用方法和函数。但这篇文章已经够长了，我们会在后续的文章里介绍这些主题。





# 我的总结
反射是Go语言的高级特性，一般情况下我们是用不到的。但在必要时，反射却很有用，可以帮助我们解决大问题，因此详细了解反射的基本规则还是很有必要的。

反射的所有能力都出自reflect包。reflect.ValueOf和reflect.TypeOf是最常用到的两个函数，当你调用这两个函数时，就打开了反射的大门，进入了反射世界。（其它的进入反射世界的还有reflect.New、reflect.MakeSlice等函数，这些具体的应用等以后的文章再介绍吧）

本篇文章主要介绍了反射的三条规则，尤其是第三条规则，对于正确使用反射来说十分重要。

前两条规则基本描述了进入和退出反射世界途径：
- 对于任何一个值，我们可以通过reflect.ValueOf生成一个代表这个值的反射对象（reflect.Value），通过这个反射对象提供的方法，使用反射的能力操控这个值。
- 对于任何一个反射对象，我们可以通过value.Interface()获取所代表的值。当然这个值只能是个interface类型，需要我们自己通过类型断言转换回真正的类型。

我们重点总结一下第三条规则。反射对象提供了一些可设置值的Set方法，通过这些方法可以修改反射对象所代表的对象的值。但并不是所有反射对象都能成功修改所代表的对象的值，其原因是一个反射对象所代表的对象永远是拷贝来的，并不是原来的对象，包括指针。比如：

```go
var x int = 3
v := reflect.ValueOf(x)
```

v所代表的并不是x，而是x的拷贝。这是因为Go中所有函数调用的传值都是拷贝的，而relfect.ValueOf也是一个函数而已。

因此，即使我们可以调用v.SetInt成功，也只不过是修改了x的拷贝的值而已，而x值仍然不变。为了避免这种初看上去正确、但非常容易隐藏bug的状况，reflect的实现里明确禁止了这种情况下对Set方法的调用（如果调用会发生panic错误）。如果想要调用Set方法，反射对象必须是可修改的（settable，即CanSet方法返回true）。

那么如何让反射对象的CanSet返回true呢？

**首先在调用reflect.ValueOf时必须传入对象的指针**。因为有了指针我们才能拿到原始对象，而不是拷贝，这个很好理解，无需多解释。比如：

```go
var x int = 3
v := reflect.ValueOf(&x)
```

此时v所代表的对象是x的指针，而不是x的拷贝了。但此时我们调用v.SetInt仍然会发生panic错误。这是为什么呢？  
前面我们说过，通过参数传来的值永远都是拷贝来的，即使是指针。那么这里反射对象v其实仍然代表了一个拷贝来的值，只不过这个值的类型是个指针。根据规则，v.CanSet当然会返回false。

（一开始我自己也有些疑惑，认为这里的v就是代表了x，并且是通过指针代表的，CanSet怎么能为false呢？仔细想想“v所代表的对象是x的指针”这句话：假如说&x的值是0x12345678，那么v所代表的值就是0x12345678，类型是\*int。也就是说，v根本没有代表x，它只是代表了一个普通的数字，数字的类型是指针。）

因此想要能修改x的值，还需进一步：**调用Elem()方法返回指针所指向的值的反射对象**。注意Elem方法与reflect.ValueOf函数不同，Elem方法返回的反射对象代表的是指针指向的对象，这是原始的对象，而非拷贝。 例如：

```go
e := v.Elem()
```

此时调用 `e.SetInt(22)` ，x的值被成功修改为22。

最后作者强调了一下结构体的修改，由于结构体有自己的字段，有些地方法需要稍微注意一下。与普通类型一样，想要修改结构体的某个字段的值，也要遵从第三条规则，但还要求字段是导出的。另外如果字段未导出，在调用反射对象的一些其它方法时（如Interface方法）也会发生失败，这在具体应用时还应多注意查看文档。

以上是我对这篇博文的翻译和总结，有不对的地方，还望大家不吝指正。
