+++
topics = [
  "topic 1",
]
keywords = [
  "golang"
]
description = "Golang的类型"
draft = true
type = "post"
date = "2016-12-11T16:15:52+08:00"
title = "Golang的类型"
tags = [
  "golang"
]
author = "asdfsx"

+++

# 类型

一种类型决定一个值的可能的取值范围以及能对这个值所支持的操作。类型有可能是已命名的(named)也可能是未命名的（unnamed）。已命名的类型会有类型名称；未命名点类型可以用`type`来定义，用已有的类型来组成新的类型。

已命名的实例：布尔、数字、字符串都是预定义的。复合类型--数组、结构体、指针、函数、接口、切片、字典、管道类型--可能需要用类型字面量来定义。


### 方法集

一个类型可能会有一个和它相关的方法集合。一个接口类型的方法集就是它的接口。其他的类型`T`的方法集是由那些接收者（参见方法定义）为`T`的方法组合而成的。指针类型`*T`的方法集是由哪些接收者是`*T`或者`T`的方法组合而成（也就是说它包含了`T`的方法集）。


常量声明
----

```
const Pi float64 = 3.14159265358979323846
const zero = 0.0         // untyped floating-point constant
const (
	size int64 = 1024
	eof        = -1  // untyped integer constant
)
const a, b, c = 3, 4, "foo"  // a = 3, b = 4, c = "foo", untyped integer and string constants
const u, v float32 = 0, 3    // u = 0.0, v = 3.0
```

变量声明
----

```
var i int
var U, V, W float64
var k = 0
var x, y float32 = -1, -2
var (
	i       int
	u, v, s = 2.0, 3.0, "bar"
)
var re, im = complexSqrt(-1)
var _, found = entries[name]
```
可以发现，变量类型跑到了变量名后边。

在__函数内部__还有一种简短的声明方式(注意是在__函数内部__)

```
i, j := 0, 10
f := func() int { return 7 }
ch := make(chan int)
r, w := os.Pipe(fd)  // os.Pipe() returns two values
_, y, _ := coord(p)
```

不需要指定类型，golang可以自己推断。

另外就是，使用这种声明方式，可能会遇到重复声明变量的问题。对此官方文档有如下说明

```
Unlike regular variable declarations, a short variable declaration may redeclare variables provided they were originally declared earlier in the same block with the same type, and at least one of the non-blank variables is new. As a consequence, redeclaration can only appear in a multi-variable short declaration. Redeclaration does not introduce a new variable; it just assigns a new value to the original.
```

例如：

```
field1, offset := nextField(str, 0)
field2, offset := nextField(str, offset)  // redeclares offset
a, a := 1, 2                              // illegal: double declaration of a or no new variable if a was declared elsewhere

```

简单说，只要里边有一个变量是之前没有用过的，就没有问题。


数据类型
----
* Boolean

布尔类型的声明和操作

```
var a bool = false
b := true 
c := a == b
c = a != b
c = !a
c = a && b
c = a || b
```

* Numeric  

数字类型如下：

```
uint8       the set of all unsigned  8-bit integers (0 to 255)
uint16      the set of all unsigned 16-bit integers (0 to 65535)
uint32      the set of all unsigned 32-bit integers (0 to 4294967295)
uint64      the set of all unsigned 64-bit integers (0 to 18446744073709551615)

int8        the set of all signed  8-bit integers (-128 to 127)
int16       the set of all signed 16-bit integers (-32768 to 32767)
int32       the set of all signed 32-bit integers (-2147483648 to 2147483647)
int64       the set of all signed 64-bit integers (-9223372036854775808 to 9223372036854775807)

float32     the set of all IEEE-754 32-bit floating-point numbers
float64     the set of all IEEE-754 64-bit floating-point numbers

complex64   the set of all complex numbers with float32 real and imaginary parts
complex128  the set of all complex numbers with float64 real and imaginary parts

byte        alias for uint8
rune        alias for int32


uint     either 32 or 64 bits
int      same size as uint
uintptr  an unsigned integer large enough to store the uninterpreted bits of a pointer value
```

变量声明及操作

```
var a int
var b uint16 = 20
var c uint32
c = uint32(b)

var c1 complex64 = 5 + 10i
```

整数支持位运算

```
1 & 1 //按位与
0 & 1 //按位与
1 | 1 //按位或
0 | 1 //按位或
1 ^ 1 //按位异或
1 ^ 0 //按位异或
1<<10 //位左移
2>>1  //位右移
```

配合iota

```
type ByteSize float64const (_ = iota // 通过赋值给空白标识符来忽略值 KB ByteSize = 1<<(10*iota)MBGBTB 
PB 
EB 
ZB 
YB)

type BitFlag intconst (    Active BitFlag = 1 << iota // 1 << 0 == 1    Send // 1 << 1 == 2    Receive // 1 << 2 == 4)flag := Active | Send // == 3
```
还有`+ - * / % ++ -- += -= *= /= %=`等常规的数值计算  
还有`> < == >= <= !=`等常规数值比较

另外字符类型可以用int来表示

```
var ch int = '\u0041'var ch2 int = '\u03B2'var ch3 int = '\U00101234'
fmt.Printf("%d - %d - %d\n", ch, ch2, ch3) // integerfmt.Printf("%c - %c - %c\n", ch, ch2, ch3) // characterfmt.Printf("%X - %X - %X\n", ch, ch2, ch3) // UTF-8 bytesfmt.Printf("%U - %U - %U", ch, ch2, ch3) // UTF-8 code point
```

* String

字符串由字节序列组成，一旦创建，它的内容就不可改变；它的长度由函数`len`获得；字符串里的字节可以用下标访问；不能获取字符串里的字节的地址,`&s[i]`是不合法的。

```
var a string = "hello"
l := len(a)
c := a[0]
```

* Array

由定长的，单一类型的数据组成的序列；长度由函数`len`获得；可以用下标访问数组元素

```
[32]byte
[2*N] struct { x, y int32 }
[1000]*float64
[3][5]int
[2][2][2]float64  // same as [2]([2]([2]float64))
```

* Slice

可以看作一个长度可变的数组；内部应该是由一个数组来实现的（官方文档总是说_an underlying array_）；长度由函数`len`获得；可以用下标访问元素；未初始化的slice的值是nil；相比数组多了一个容量的概念，可以由函数`cap`获得。  
创建slice要用`make`函数

```
make([]T， length)
make([]T， length， capacity)
```
分片通常也是一维的，但是也可以组合出更高维的对象。对于数组的数组来说，内层的数组的长度在构造的时候总是一样的；然而分片的分片(分片的数组)的长度却是可以变化的。此外，内层的分片需要单独创建。

* Struct

由多个字段组成

```
// An empty struct.
struct {}

// A struct with 6 fields.
struct {
	x, y int
	u float32
	_ float32  // padding
	A *[]int
	F func()
}
```

字段可以匿名

```
// A struct with four anonymous fields of type T1, *T2, P.T3 and *P.T4
struct {
	T1        // field name is T1
	*T2       // field name is T2
	P.T3      // field name is T3
	*P.T4     // field name is T4
	x, y int  // field names are x and y
}
```



一个字段的声明中可以跟着一个可选的字符串标签，它在相应的字段声明中会算做字段的一种属性/性质。这些标签在反射接口和类型一致那里中是可见的，其他的时候可以认为是忽略不计的。

```
// 一个用于时间戳协议缓冲区的结构体
// 标签字符串定义了协议缓冲区字段号
struct {
  microsec  uint64 "field 1"
  serverIP6 uint64 "field 2"
  process   string "field 3"
}
```

* Pointer

指针不多说，未初始化的指针值是nil。

```
var p *Point
var pp *[4]int
```

* Function

描述一类函数的特征，未初始化的函数变量的值是nil。  
在函数的参数/结果列表中，名字（标识符列表）可以都有也可以没有。如果有的话，一个名字代表对应类型的一项（参数/结果），非空名称必须不相同；如果没有，一个类型代表该类型的一项。参数/结果列表通常用小括号括起来，不过当只有一个返回值且没有名字的情况下，这个括号可以省略掉。  
一个函数签名的最后一个参数可能以...为前缀，这样的函数我们叫可变函数，它们在调用的时候对于那个参数可以传递0或是多个值。


```
func()
func(x int) int
func(a, _ int, z float32) bool
func(a, b int, z float32) (bool)
func(prefix string, values ...int)
func(a, b int, z float64, opt ...interface{}) (success bool)
func(int, int, float64) (float64, *[]int)
func(n int) func(p *T)
```

* Interface

接口由一组不重名的函数组成
```
// A simple File interface
interface {
	Read(b Buffer) bool
	Write(b Buffer) bool
	Close()
}
```


* Map

Map是无序的；为初始化的Map值是nil；长度由函数`len`获得；删除元素由函数`delete`完成。  
key的类型必须完整的实现了`comparison operators == and !=`, 所以key不能是`function, map, or slice`。  

```
map[string]int
map[*T]struct{ x, y float64 }
map[string]interface{}
```

创建map要用`make`函数

```
make(map[string]int)
make(map[string]int， 100)
```

Map创建后，容量没有限制。

* Channel

管道用来实现并发编程中的通信。为初始化的管道的值是nil。

```
chan T          // can be used to send and receive values of type T
chan<- float64  // can only be used to send float64s
<-chan int      // can only be used to receive ints
```

初始化管道要用函数`make`

```
make(chan int, 100)
```

* 自定义类型