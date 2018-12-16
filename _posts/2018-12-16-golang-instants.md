---
layout: post
title:  "golang常量"
categories: golang
tags: golang 翻译 constants
author: fatcat22
---

* content
{:toc}




本文是对Go Blog上[一篇文章](https://blog.golang.org/constants)的翻译和自己的简单总结。

# **翻译**

### **引言**
Go是一门静态类型的语言，并且不允许对不同类型的数字进行运算。比如你不能将一个float64类型的变量和一个int类型的变量相加，甚至不能将一个int32类型的和int类型的相加。然而像1e6*time.Second或math.Exp(1)甚至1<<('\t'+2.0)这些写法却是合法的。在go语言中，与变量比起来，常量(constants)的表现更符合我们对常规数字的认知。这篇文章解释了这其中的原因以及意义。

### **背景：C语言**
在早期构思Go的日子里，我们讨论了许多由于允许将不同的数字类型混合在一起操作而引起的问题，这些问题主要存在于C语言及衍生的语言中。许多奇怪的bug、崩溃和移植性问题都是由将不同长度和符号（正数和负数）组合在一起的表达式引起的。虽然对于一个C语言老手来说下面的计算可能很熟悉：
```c
unsigned int u = 1e9;
long signed int i = -1;
... i + u ...
```
但其结果却不是一眼就能看出来：运算结果的长度是多长？值是多少？这个值是有符号还是无符号的？

可恶的bug潜伏其中。

C语言有一系列称为“算术转换惯例”的规则。随着这些规则多年来发生的变化，它们已经变成了C的微妙之处的一种标志（从而引进了更多的bug）。

当我们设计Go时，我们决定通过强制禁止将不同类型的数字混合在一起操作来避免这一“雷区”。如果你想将变量i和u相加，你必须对你想要的结果进行明确的定义。给定变量
```go
var u uint
var i int
```
你可以写成uint(i)+u或i+int(u)，每一种写法都是一个含义和类型都很明确的加法表达式。但不像C，你不能写成i+u。你甚至不能将int和int32混在一起，即使int是32位的。

这一严格的限制消除了一些常见的bug和问题，是Go的至关重要的一个特性。但这样做也有代价：有时候这样做会让程序员不得不写出看上去非常笨拙的类型转换的代码。

那么常量需要类型转换吗？我们认为让常量进行类型转换是很荒唐的，比如i = int(0)。但 给定上面的变量声明，为什么i=0和u=0都是合法的？0到底是什么类型呢？

我们很快就会发现问题的答案就存在于不同于类C语言的常量机制。经过反复的思考和试验，我们认为避免让程序员对常量进行类型转换从而能够写出如math.Sqrt(2)这样的代码是非常正确的。

简单来说，常量在Go中大多数情况下都工作的非常好。我们这就来看看具体细节。

### **名词解释**
首先我们来一个快捷的定义。在Go里，const是一个关键字，它用来将一个值进行命名，比如2或者3.14159或"scrumptions"。像这样的值无论是有名字或没有名字的，被称为常量（cconstants）。常量也可以由常量组成的表达式创建，比如2+3或math.Pi/2或("go"+"pher")。

有些编程语言没有常量，有些语言中常量有更广泛的含义。比如在C和C++中，const是一个可以规定更多属性或值的修饰词（比如const int*和int* const）。

但在Go语言中，一个常量仅仅就是一个简单的、不可改变的值。从现在开始，我们只讨论Go中的常量。

### **字符串常量**[\[1\]](#comment_1)
Go语言里有很多种数字常量：整数、浮点数、字符、有符号的、无符号的、虚数、复数。所以我们先从简单一点的常量开始：字符串常量。字符串常量非常容易理解，因此在整个常量讨论中占的篇幅相对较少。

一个字符串常量就是由双引号引起来的一串字符（Go中也有raw字符串，但对于这里的讨论它们都是一样的，没有区别）。下面是一个字符串常量：
```
"hello, 世界"
```
（关于字符串更多的信息，可以看一下[这篇文章](https://blog.golang.org/strings)）

这个字符串的类型是什么呢？显而易见的答案是string类型，但这是错的。

这是一个**untyped字符串常量**，也就是说一个没有固定类型的文本值。是的它是一个字符串，但它不是一个Go语言中的string。即使给了一个名字也一样：
```go
const hello = "Hello, 世界"
```
这条声明之后，hello也是一个untyped字符串常量。一个untyped常量只是一个值，这个值不会被设定一个特定的类型，因而也不会因为严格的类型规则而不能与不同的类型组合。

正是这个“untyped常量”的概念使我们在Go中非常自由的使用常量。

那么什么是一个typed字符串常量呢？一个typed字符串常量是一个给定了类型的常量，比如：
```go
const typedHello string = "Hello, 世界"
```
注意typedHello的声明在等号前带有一个明确的类型 string。这意味着typedHello的类型就是Go的string，并且不能使用其它类型的变量进行赋值。也就是说下面的代码是正确的：
```go
var s string
s = typedHello
fmt.Println(s)
```
但下面的代码是错误的：
```go
type MyString string
var m MyString
m = typedHello //Type error
fmt.Println(m)
```
变量m的类型为MyString并且不能使用其它类型的值赋值，只能使用MyString类型的值。比如：
```go
const myStringHelo MyString = "Hello, 世界"
m = myStringHello //OK
fmt.Println(m)
```
或者进行强制类型转换，比如：
```go
m = MyString(typeHello)
fmt.Println(m)
```

现在我们回到untyped字符串常量上来。untyped字符串常量有一个非常有用的特性，那就是因为它没有类型，所以可以合法地赋给一个具有特定类型的变量。比如我们可以这样写：
```go
m = "Hello, 世界"
```
或者
```go
m = hello
```
因为不像typed常量typedHello和myStringHello，untyped常量"Hello，世界"和hello没有类型，把它们赋给任意一个和字符串类型兼容的变量不会引起任何错误。

当然了，这些untyped字符串常量是一些字符串，所以它们只能在允许使用字符串的地方使用，但它们不具有Go中的string类型。

### **默认类型**
作为一个Go程序员，你肯定见过许多这样的声明：
```go
str := "Hello, 世界"
```
那么现在你可能会问，如果这是一个untyped常量，在这个声明中str是如何获得一个类型的呢？答案就是untyped常量有一个默认的类型，当没有为一个变量显式提供类型定义时，这个默认的类型就是这个变量的类型。对于untyped字符串常量来说，它的默认类型就是string。所以：
```go
str := "Hello, 世界"
```
或
```go
var str = "Hello, 世界"
```
和下面的声明的含义相同
```go
var str string = "Hello, 世界"
```

一种理解untyped常量的方式是你可以认为它们存在于一种值空间中，这个空间的类型规则的限制要比Go的少得多。但当我们要使用常量来做什么事时，我们需要把它们赋给一些变量，这时这些变量需要一个类型，而常量可以告诉这些变量应该是什么类型。在上面的例子中，str成为了一个string类型的变量，因为untyped字符串常量把它的默认类型string给了它。

在上面的例子中，声明str变量后，str拥有了类型和初始值。但有时候我们使用常量时，目标值并不总是这么明确。例如下面这个语句：
```go
fmt.Printf("%s", "Hello, 世界")
```
其中fmt.Printf函数的声明为
```go
func Printf(format string, a ...interface{}) (n int, err error)
```
也就是说它fmtart之后的参数是interface值。当使用untyped常量作为参数调用fmt.Printf时，一个interface类型的值被创建出来并作为参数传入，而这个interface类型的值指向的具体类型则是这个常量的默认类型。这个处理过程和我们前面例子中声明变量并使用untyped字符串常量初始化的过程是类似的。

下面的例子使用%v打印参数的值并且使用%T打印参数的类型：
```go
fmt.Printf("%T: %v\n", "Hello, 世界", "Hello, 世界")
fmt.Printf("%T: %v\n", hello, hello)
```
输出结果为：
>string: Hello, 世界  
>string: Hello, 世界

如果将一个有具体的类型的常量作为参数传入：
```go
fmt.Printf("%T: %v\n", myStringHello, myStringHello)
```
输出：
>main.MyString: Hello, 世界

(更多关于interface相关的信息，可以看[这篇文章](https://blog.golang.org/laws-of-reflection))

总得来说，typed常量遵守所有拥有具体类型的变量的类型规则。另一方面，untyped常量却没有一个具体的Go里的类型因而可以更自由的和组合和使用。但它确实有一个的类型，当且只当没有其它类型信息时会被使用。

### **通过写法判别默认类型(Default type determined by syntax)**[\[2\]](#comment_2)
一个untyped常量的默认类型是通过它的写法来判断的。对于字符串常量来说，唯一可能隐含的类型就是string。对于数字常量来说，隐含的类型要更加多样。整数常量默认是int，浮点数常量默认是float64，字符常量默认是rune（int32的别名），复数常量默认是complex128。下面是我们使用显示默认类型的典型语句：
```go
fmt.Printf("%T %v\n", 0, 0)
 fmt.Printf("%T %v\n", 0.0, 0.0)
 fmt.Printf("%T %v\n", 'x', 'x')
 fmt.Printf("%T %v\n", 0i, 0i)
```
输出:
>int 0  
>float64 0  
>int32 120  
>complex128 (0+0i)

### **Booleans**
untyped布尔常量符合我们前面描述的所有untyped字符串常量的信息。true和false都是untyped布尔常量，可以将它们赋给任意的布尔变量，但一旦给定的类型，布尔变量不能和其它类型混合：
```go
type MyBool bool
const True = true
const TypedTrue bool = true
var mb MyBool
mb = true      // OK
mb = True      // OK
mb = TypedTrue // Bad
fmt.Println(mb)
```

### **Floats**
浮点数常量和布尔常量在很多方面都很相似：
```go
type MyFloat64 float64
const Zero = 0.0
const TypedZero float64 = 0.0
var mf MyFloat64
mf = 0.0       // OK
mf = Zero      // OK
mf = TypedZero // Bad
fmt.Println(mf)
```
一个不同的地方在于在Go语言中有两种类型的浮点数：float32和float64。浮点数常量的默认类型是float64，但一个untyped浮点数常量依然可以赋值给float32类型的变量：
```go
var f32 float32
f32 = 0.0
f32 = Zero      // OK: Zero is untyped
f32 = TypedZero // Bad: TypedZero is float64 not float32.
fmt.Println(f32)
```

现在是一个很好的时机引出值溢出这个概念。

数字常量在一个任意精度的数字空间中，它们只是平时我们所认识的数字罢了。但当将它们赋给一个变量时，它们必须适配这个变量。比如我们可以声明一个非常大的数字常量：
```go
const Huge = 1e1000
```
这只是一个数字罢了。但我们不能将它赋给某个变量甚至是打印它。下面的语句都不会通过编译：
```go
fmt.Println(Huge)
```
编译时错误信息为：constant 1e+10000 overflows float64。但Huge也可以很有用：我们可以将它在一个表达式中和其它常量一起使用，如果这个表达式的值在float64类型的范围内，我们就可以使用这个值。比如：
```go
fmt.Println(Huge / 1e999)
```
如我们所期望的，这条语句输出10。

类似的，浮点数常量拥有非常高的精度，因此使用这些常量的计算精度更高。math包中定义的一些常量拥有比float64类型可以拥有的更多的位数。比如math.Pi的定义：
```go
Pi    = 3.14159265358979323846264338327950288419716939937510582097494459
```
当这个常量值赋值给一个变量时，会丢失一些精度。这类赋值会创建一个精度最接近常量的精度的float64(或float32)变量。比如下面的语句
```go
pi := math.Pi
fmt.Println(pi)
```
会输出3.141592653589793

拥有如此多的有效位意味着如形如Pi/2或更加复杂的运算可以一直保持非常高的精度，直到其结果被赋值给一个变量，这也让使用常量的运算在不丢失精度的情况下更加容易实现。另外还意味着使用常量表达式计算时不会发生像浮点数变量计算时可能出现的像无穷大（除0是一个编译时错误）、非数字错误（NaN，当运算的所有的元素都是数字时不会发生如“不是一个数字”的错误）等问题。

### **Complex numbers**
复数常量和浮点数常量非常类似。下面的例子现在对我们来说已经非常熟悉了：
```go
type MyComplex128 complex128
const I = (0.0 + 1.0i)
const TypedI complex128 = (0.0 + 1.0i)
var mc MyComplex128
mc = (0.0 + 1.0i) // OK
mc = I            // OK
mc = TypedI       // Bad
fmt.Println(mc)
```
复数常量的默认类型是complex128。complex128是由两个float64值组合而成的，具有更大的精度。

为了让示例更清析，我们使用了完整的表达式(0.0+1.0i)，但实际上它可以简写成0.0+1.0i，1.0i，甚至1i。

接下来我们来看一个小窍门。我们知道在Go中，一个数字常量仅仅是一个数字而已。那么如果这个数字是一个没有虚部的复数呢？它会是一个实数吗？比如：
```go
const Two = 2.0 + 0i
```
这是一个untyped复数常量。即使它没有虚部，这种写法仍然让它使用了complex128作为默认类型。因此我们可以使用它定义一个变量，这个变量的类型将会是complex128：
```go
s := Two
fmt.Printf("%T: %v\n", s, s)
```
输出为：
>complex128: (2+0i)

但从数值上说，常量Two可以使用浮点数float64或float32存储，并且不会丢失任何信息。因此我们可以将Two正常赋值给一个float64变量：
```go
var f float64
var g float64 = Two
f = Two
fmt.Println(f, "and", g)
```
输出：
>2 and 2

所以即使Two是一个复数常量，但它仍然可以赋值给浮点数变量。常量的这种“跨类型”的能力是非常有用的。

### **Intergers**
最后我们来讨论整数。整数有更多的变数：多种长度（8位，32位，64位）、有无符号等等[\[3\]](#comment_3)。但它们遵守相同的规则。我们最后一次使用已经非常熟悉的例子，只是将类型改成了int:
```go
type MyInt int
const Three = 3
const TypedThree int = 3
var mi MyInt
mi = 3          // OK
mi = Three      // OK
mi = TypedThree // Bad
fmt.Println(mi)
```
这个例子也可以改成任意其它的整数类型，包括：
>int int8 int16 int32 int64  
>uint uint8 uint16 uint32 uint64  
>uintptr  

再加上uint8和int32的别名byte和rune，虽然有很多，但它们在常量的性质上是一样的。

正如前面提到的，整数分成了几组形式，每组都有它自己的默认类型：简单的像123或0xff或-14的默认类型是int，字符常量像'a'或'世'或'\r'的默认类型是rune。

没有常量以无符号整数类型作为它的默认类型，但untyped常量的灵活性意味着我们可以使用整数常量初始化类型为无符号整数的变量，这与我们使用虚部为0的复数初始化一个float64类型的变量类似。下面是几种初始化uint的不同方式，它们都是等价的，但每一种都必须明确的写明无符号类型：
```go
var u uint = 17
var u = uint(17)
u := uint(17)
```
与前面讨论浮点数常量时提到的溢出问题类似，不是所有整数常量都可以赋值给Go的整数类型。主要有两个问题会导致赋值失败：一是整数常量太大；二是赋值时将一个负数常量赋给一个无符号类型的变量。比如int8类型的变量数值范围是-128到127，所以超出这个范围的整数常量无法赋值给int8类型的变量：
```go
var i8 int8 = 128 // Error: too large.
```
类似的，uint8或者byte的数值范围是0到255，所以一个太大的或负数常量不能赋值给一个uint8变量：
```go
 var u8 uint8 = -1 // Error: negative value.
```
编译器也会捕捉到这样的错误：
```go
type Char byte
var c Char = '世' // Error: '世' has value 0x4e16, too large.
```
如果编译器在你使用常量时报错，一般情况下这可能真的是一个bug。

### **一个练习：unsigned int的最大值**
这里我们来一个需要很多常量知识的小练习。我们如何使定义一个代表uint类型最大值的常量？如果我们讨论的是uint32类型而非uint，那么我们可以这样写：
```go
const MaxUint32 = 1<<32 - 1
```
但我们讨论的是uint。int和uint类型的位数相同，都是32位或者64位。因为它们的位数根据所使用的操作系统构架（architecture）不同而不同，所以我们不同简单的写一个固定的值。

Go的整数运算遵守[补码](https://en.wikipedia.org/wiki/Two's_complement)的规则。了解补码的同学都知道，-1的所有位都是1，所以-1的各个位本质上和最大的正整数是一样的。所以我们可能可能这样写：
```go
const MaxUint uint = -1 // Error: negative value
```
但这样写是错误的，因为一个无符号变量不能表示-1；-1不在无符号数值的数值范围之内。出于同样的原因，加上类型转换也不行：
```go
const MaxUint uint = uint(-1) // Error: negative value
```
即使在运行时-1可以被转换成一个无符号整数，但编译时却不行，常量的转换规则会阻止这种操作。也就是说，下面的代码可以正常运行：
```go
var u uint
var v = -1
u = uint(v)
```
但仅仅是因为v是一个变量；如果v是一个常量，即使是一个untyped常量，仍然会编译失败：
```go
var u uint
const v = -1
u = uint(-1) // Errr: negative value
```
回到我们之前的思路上，但这次我们尝试使用^0（对0进行按位非操作）代替-1。但这仍然会报错。因为从数值的角度上看，^0代表了一个位数无限且每一位上都是1的数值，所以将这个值赋给一个固定长度的类型时就会出错：
```go
const MaxUint uint = ^0 // Error: overflow
```
那么我们到底要如何才能写一个代表uint类型的最大值的常量呢？

经过上面的讨论，我们认识到有两个关键点：一是约束数值的位数不能超过uint类型的位数；二是要避免将负数赋值给uint类型。满足第一点的要求的最简单的方法是typed常量uint(0)；满足第二点的方法我们已经讨论过，使用^0代替-1。综合这两点，如果我们将uint(0)的所有位取非，我们就得到了正确的结果：
```go
const MaxUint = ^uint(0)
```
无论当前的执行环境中uint类型的位数是多少，这个常量都可以正确的代表uint类型的最大值。

如果你能理解我们得到这个答案的所有分析，你就理解了Go中关于常量的所有要点。

### **Numbers**
Go中untyped常量的概念意味着，所有的数字常量，无论是整数、浮点数、复数，还是字符，都是同一个概念。只有把这些常量带到有变量、赋值或其它操作的计算世界时，才产生了区别。但只要在常量的世界里，我们可以任意混合使用这些常量。下面的所有常量都是数值1：
>1  
>1.000  
>1e3-99.0*10-9  
>'\x01'  
>'\u0001'  
>'b' - 'a'  
>1.0+3i-3.0i

因此，虽然它们拥有不同的默认类型，但untyped常量可以赋值给任意数字类型的变量：
```go
var f float32 = 1
var i int = 1.000
var u uint32 = 1e3 - 99.0*10.0 - 9
var c float64 = '\x01'
var p uintptr = '\u0001'
var r complex64 = 'b' - 'a'
var b byte = 1.0 + 3i - 3.0i

fmt.Println(f, i, u, c, p, r, b)
```
这代代码将输出：1 1 1 1 1 (1+0i) 1

你甚至可以写出这样古怪的代码：
```go
var f = 'a' * 1.5
fmt.Println(f)
```
这将会输出145.5。虽然这看上去应该没什么意义。

但是注意，这里真正的意义是灵活性。常量的灵活性意味着尽管Go不允许在同一个表达式里混合使用浮点数和整数变量，甚至不允许int和int32变量，但这样写却没有问题（math.Sqrt的参数类型是float64）：
```go
sqrt2 := math.Sqrt(2)
```
或者
```go
const millisecond = time.Second/1e3
```
或者
```go
bigBufferWithHeader := make([]byte, 512+1e6)
```
以上写法都是正确的。

因为在Go里，数值常量和你所期望的一样：它们就是数值而已。



# **我的总结**
Go语言里面有严格的变量类型限制，不同类型不能放在同一个表达式进行运算，即使是int和uint也是如此。这种严格的限制消除了潜在的产生bug的机会，但有时也让代码看上去不那么美观。但我认为Go的许多地方都有这种取舍：宁愿用笨笨的写法，而让代码准确性和易理解性更高。

但对于常量为说，如果也要这样限制就显得太笨了。所以Go的设计者在这个地方决定即要保证代码的直观性，也要保证正确性，因此有了Go中关于常量的设计。

Go里面的常量分三类：字符串，布尔，数字。与变量不同的是，它们没有固定的类型。怎么理解呢？想像一下你在不知道什么是编程的时候，看到一个字符串，你会想到它是string类型吗（对于C语言开发者这里应该问你会想到它是char*类型吗）？显然不会，你只知道这是一串文本而已；看到一个数字比如1、2.1，你会想到它们是int或byte或是float64吗？也不会，在非编程的世界里它们都是数字。Go里面的常量也应该这么理解。

这就是所谓的untyped常量。

这种untyped常量的设计带来了很大的方便，在使用时可以以我们常规的认知使用常量。比如:
```go
var x int = 1.0

type MyString string
var ms MyString = "hello"
```

untyped常量虽然没有固定的类型，但都会有一个默认类型。当在一个表达式中无法判断应该使用什么类型时，常量的默认类型就生效了。比如：
```go
f := 2.1
```
这里没有任何信息可以判断变量f的类型，只能使用2.1的默认类型。

各个常量的默认类型是通过常量的字面写法来决定的，比如2.1这一看就是个浮点数，"hello"一看就是个字符串。默认类型的对应关系总结如下：

常量类型 | Go中的类型
---------|-----
字符串 | string
布尔  | bool
整数 | int
浮点数 | float64
复数  | complex128
字符  | rune

对于数字类型的常量需要多注意几点。

一是溢出问题。由于数字常量就是我们日常所认知的数字，没有32位64位的限制，因此你可以写一个巨大无比的数字常量，这是没有问题的。但当你把它赋值给某个具体类型的变量时，如果这个变量的类型无法存放这么大的数值，编译器就会报错。比如：
```go
const Huge = 0xffffffff0
var x uint64 = Huge //Error
```
另外，当把一个负数赋给一个无符号类型时，也会报错：
```go
var x uint = -1
```
就这个例子来说，-1作为一个纯粹的数字，超出来无符号类型uint所能代表的范围。

二是数值常量真的就是一个数值而已。在数学里，你能说2.0不等于2吗？实际上它不仅等于2，而且就是2。对于数值常量来说也是如此。因此下面的语句是正确的：
```go
var x int = 2.0
var x float64 = 2 + 0i
```
同样的原因，浮点数常量可以拥有比Go里float64大得多的精度，比如math.Pi。


-------------
注释：

<a id="comment_1"/>
[1]  在这篇翻译的文章里，我使用中文“字符串”或“字符串常量”指代由一系列字符组成的一段文本；使用英文"string"指代Go语言中的string类型。

<a id="comment_2"/>
[2]  原文是"Default type determined by syntax"。我觉得"syntax"在此处翻译成“语法”容易产生误解，让人以为是编译语言的那种“语法”。实际上这里作者想表达的意思应该是按常量的字面意思来决定默认类型，我这里翻译成了“写法”。

<a id="comment_3"/>
[3]  其它变数包括int类型在不同系统下有不同的位数；uintptr未明确定义位数。
