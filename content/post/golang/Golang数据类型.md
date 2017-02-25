+++
topics = [
  "topic 1",
]
keywords = [
  "golang"
]
description = "Golang的类型"
draft = false
type = "post"
date = "2017-02-25T20:15:52+08:00"
title = "Golang的类型"
tags = [
  "golang"
]
author = "asdfsx"

+++

两个月前给自己挖的坑，就这么算是填上了吧。其实最重要的还是多看代码、多写代码。本文参考 Golang Specification 和 The way to go。

# 常量
### 常量声明

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
### Iota
```
const (
	Sunday = iota
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Partyday
	numberOfDays  // this constant is not exported
)

const ( // iota is reset to 0
	a = 1 << iota  // a == 1
	b = 1 << iota  // b == 2
	c = 3          // c == 3  (iota is not used but still incremented)
	d = 1 << iota  // d == 8  (注意这里的 << 是在进行位移运算)
)

const ( // iota is reset to 0
	c0 = iota  // c0 == 0
	c1 = iota  // c1 == 1
	c2 = iota  // c2 == 2
)



const ( // iota is reset to 0
	u         = iota * 42  // u == 0     (untyped integer constant)
	v float64 = iota * 42  // v == 42.0  (float64 constant)
	w         = iota * 42  // w == 84    (untyped integer constant)
)

const x = iota  // x == 0  (iota has been reset)
const y = iota  // y == 0  (iota has been reset)

```

在同一个表达式中，iota 的值相同，因为只有在一个常量声明表达式结束之后，它才会增加
```
const (
	bit0, mask0 = 1 << iota, 1<<iota - 1  // bit0 == 1, mask0 == 0
	bit1, mask1                           // bit1 == 2, mask1 == 1
	_, _                                  // skips iota == 2
	bit3, mask3                           // bit3 == 8, mask3 == 7
)
```

# 变量
变量就是一个用来保存值的存储空间。其中能存放的值与变量的类型一致。

在函数声明时可以为函数参数和返回值定义变量，这会为已命名的变量预留存储空间。调用内建函数`new`或者使用复合类型的地址可以在运行时获得存储空间。这样的匿名变量是通过指针直接或间接使用的。

结构化类型的变量，如数组、切片和结构体，都包含多个元素或者字段，它们的地址都是独立的。每个元素就像变量一样。

变量的静态类型在声明的时候定义，也可以通过一下的方式提供：通过调用`new`或者复合字面量，或者结构化类型中元素的类型。接口类型的变量同样也有独特的动态类型，就是在运行时分配给变量的值的类型（预定义的标识符`nil`除外,因为它没有类型）。动态类型可能会在计算时发生变化，但是存储在接口变量中的值总是可以赋给变量的静态类型。

```
var x interface{}  // x is nil and has static type interface{}
var v *T           // v has value nil, static type *T
x = 42             // x has value 42 and dynamic type int
x = v              // x has value (*T)(nil) and dynamic type *T
```

在表达式中，变量的值可以通过变量来获得；它只是最后一次赋予变量的值。如果一个变量还没有被赋值，它的值就是所属类型的`0`值。

### 变量声明

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

### 简短变量声明
在 __函数内部__ 还有一种简短的声明方式(注意是在 __函数内部__ )

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

这种声明方式也可以使用在 `if`, `for`, `switch`语句的初始化部分，生成的变量只能在这几种语句的内部使用（也就是作用域范围内）
```
if test, ok := map[string]string{"test":"test"}["test"]; ok {
	fmt.Println(test)
}
```

# 类型

一种类型决定一个值的可能的取值范围以及能对这个值所支持的操作。类型有可能是已命名的(named)也可能是未命名的（unnamed）。已命名的类型会有类型名称；未命名点类型可以用`type`来定义，用已有的类型来组成新的类型。

```
Type      = TypeName | TypeLit | "(" Type ")" .
TypeName  = identifier | QualifiedIdent .
TypeLit   = ArrayType | StructType | PointerType | FunctionType | InterfaceType |
	    SliceType | MapType | ChannelType .
```

已命名的实例：布尔、数字、字符串都是预定义的。复合类型--数组、结构体、指针、函数、接口、切片、字典、管道类型--可能需要用类型字面量来定义。


###  方法集

一个类型可能会有一个和它相关的方法集合。一个接口类型的方法集就是它的接口。其他的类型`T`的方法集是由那些接收者（参见方法定义）为`T`的方法组合而成的。指针类型`*T`的方法集是由哪些接收者是`*T`或者`T`的方法组合而成（也就是说它包含了`T`的方法集）。

### Boolean

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

### Numeric  

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

__变量声明及操作__

    ```
    var a int
    var b uint16 = 20
    var c uint32
    c = uint32(b)

    var c1 complex64 = 5 + 10i
    ```

__整数支持位运算__

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

__配合iota__

    ```
    type ByteSize float64    const (        _ = iota // 通过赋值给空白标识符来忽略值 KB ByteSize = 1<<(10*iota)        MB        GB        TB 
        PB 
        EB 
        ZB 
        YB    )

    type BitFlag int    const (        Active BitFlag = 1 << iota // 1 << 0 == 1        Send // 1 << 1 == 2        Receive // 1 << 2 == 4    )    flag := Active | Send // == 3
    ```
    
还有`+ - * / % ++ -- += -= *= /= %=`等常规的数值计算  
还有`> < == >= <= !=`等常规数值比较

另外字符类型可以用int来表示

    ```
    var ch int = '\u0041'    var ch2 int = '\u03B2'    var ch3 int = '\U00101234'
    fmt.Printf("%d - %d - %d\n", ch, ch2, ch3) // integer    fmt.Printf("%c - %c - %c\n", ch, ch2, ch3) // character    fmt.Printf("%X - %X - %X\n", ch, ch2, ch3) // UTF-8 bytes    fmt.Printf("%U - %U - %U", ch, ch2, ch3) // UTF-8 code point
    ```

### String

字符串由字节序列组成，一旦创建，它的内容就不可改变；它的长度由函数`len`获得；字符串里的字节可以用下标访问；不能获取字符串里的字节的地址,`&s[i]`是不合法的。

__变量声明与操作__

    ```
    var a string = "hello"
    l := len(a)
    c := a[0]
    s:= "hel" + "lo"
    s += " world!"
    ```
    
更多的字符串操作可以通过包`strings`和`strconv`来实现。
    
    ```
    strings.HasPrefix(s, prefix string) bool
    strings.HasSuffix(s, prefix string) bool
    strings.Contains(s, substr string) bool
    strings.Index(s, str string) int
    strings.LastIndex(s, str string) int
    strings.IndexRune(s string, r, rune) int
    strings.Replace(str, old, new, n) string
    strings.Count(s, str string) int
    strings.Repeat(s, count int) string
    strings.ToLower(s) string
    strings.ToHigher(s) string
    strings.TrimSpace(s string) string
    strings.Trim(s string, trim string) string
    strings.TrimLeft
    strings.TrimRight
    strings.Fields(s) []string
    strings.Split(s, sep) []string
    strings.Join(sl []string, sep string) string
    strings.NewReader(s string) Reader
    
    strconv.IntSize()
    strconv.Itoa(i int) string
    strconv.Atoi(s string) (i int, err error)
    strconv.FormatFloat(f float64, fmt byte, prec int, bitsize int) string
    strconv.ParseFloat(s string, bitsize int) (f float64, err error)
    ```

### Array

由定长的，单一类型的数据组成的序列；长度由函数`len`获得；可以用下标访问数组元素

```
[32]byte
[2*N] struct { x, y int32 }
[1000]*float64
[3][5]int
[2][2][2]float64  // same as [2]([2]([2]float64))
```

__变量声明与操作__

```
var arr1 [5]int
var arr2 = new([5]int) (数组是值类型，所以可以用new来创建，获得一个指向[5]int的指针)
var arr3 = [5]int{1,2,3} ==> [1,2,3,0,0]
var arr4 = [...]int{1,2,3} ==> 切片[1,2,3]
var arr5 = [5]string{3:"Chris", "4":"Ron"} ==> ["","","","Chris","Ron"]
length := len(arr1)
captis := cap(arr1) (在数组中，len、cap的结果是一样的)
```

__数组的遍历__

```
for i:= 0; i < len(arr1); i++ {
	arr[i] = ...
}

for i, value := range arr1{
	arr1[i] == value
}
```

__多维数组__

```
var screen [800][600]int

for i:= 0; i<len(screen);i++{
    for j:= 0;j<len(screen[i]);i++{
        fmt.Println(screen[i][j])
    }
}
```

### Slice

可以看作一个长度可变的数组；内部应该是由一个数组来实现的（官方文档总是说_an underlying array_）；长度由函数`len`获得；可以用下标访问元素；未初始化的slice的值是nil；相比数组多了一个容量的概念，可以由函数`cap`获得。  

__变量的声明__
```
var identifer []type (初始化之前 identifer 的值是 nil)
var idenrifer []type = arr1[start:end]
var idenrifer []type = arr1[start:]
var idenrifer []type = arr[:end]
var idenrifer []type = arr[:]

var identi []int = []int{1,2,3,4,}
```

__使用`make`函数创建slice__
```
t := make([]int， 5) => [0,0,0,0,0] (cap为5切片，并且切片里的值全部设为int的默认值)
t := make([]int， 3， 5) => [0,0,0] (cap为5的切片，并且其中的3个设为int的默认值)

注意：
使用 make([]int， 5) 创建的slice，已经被 0 填满。这时使用 append 添加新数据的时候，会加在slice的末尾。
所以如果想用 make 预先分配一些空间出来，需要写成 make([]int, 0, 5)
```

__切片的复制和追加__
```
var from = []int{1,2,3}
var to = make([]int, 0)

n := copy(to, from)

to = append(to, 1, 2, 3)

```


分片通常也是一维的，但是也可以组合出更高维的对象。对于数组的数组来说，内层的数组的长度在构造的时候总是一样的；然而分片的分片(分片的数组)的长度却是可以变化的。此外，内层的分片需要单独创建。

__切片和垃圾回收__

切片的底层指向一个数组，该数组的实际体积可能要大于切片所定义的体积。只有在没有任何切片指向的时候，底层的数组内层才会被释放，这种特性有时会导致程序占用多余的内存。

示例 函数 FindDigits 将一个文件加载到内存，然后搜索其中所有的数字并返回一个切片。

``` 
var digitRegexp = regexp.MustCompile("[0-9]+")
func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
} 
```
 
这段代码可以顺利运行，但返回的 []byte 指向的底层是整个文件的数据。只要该返回的切 片不被释放，垃圾回收器就不能释放整个文件所占用的内存。换句话说，一点点有用的数据 却占用了整个文件的内存。 
想要避免这个问题，可以通过拷贝我们需要的部分到一个新的切片中:

```
func FindDigits(filename string) []byte {
   b, _ := ioutil.ReadFile(filename)
   b = digitRegexp.Find(b)
   c := make([]byte, len(b))
   copy(c, b) 
   return c 
}  
```

### Map
Map是引用类型；未初始化的Map值是nil；长度由函数`len`获得；删除元素由函数`delete`完成；Map是无序的。

key的类型必须完整的实现了`comparison operators == and !=`, 所以key不能是`function, map, or slice`。  

```
map[string]int
map[*T]struct{ x, y float64 }
map[string]interface{}
```

变量的声明
```
var map1 map[string]string
```

创建map要用`make`函数
```
a := make(map[string]int)
b := make(map[string]int， 100)
b := make(map[string][]int， 100)
```

Map创建后，容量没有限制。

### Struct

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

也可以将一个 struct 嵌入到另外一个struct 中

```
type A struct{
	v int
}
type B struct {
	A
}
```

这时可以用 b.v 来访问 a 中的元素，这叫做 promoted。


一个字段的声明中可以跟着一个可选的字符串标签，它在相应的字段声明中会算做字段的一种属性/性质。这些标签在反射接口和类型一致那里中是可见的，其他的时候可以认为是忽略不计的。

```
// 一个用于时间戳协议缓冲区的结构体
// 标签字符串定义了协议缓冲区字段号
import reflect
import fmt

type Tagtest struct {
  microsec  uint64 "field 1"
  serverIP6 uint64 "field 2"
  process   string "field 3"
}

tt := &Tagtest{}
ttType := reflect.TypeOf(tt)
for i:= 0;i<ttType.NumField();i++{
    ixField := ttType.Field(i)
    fmt.Println(ixField.Tag)
}
```


__创建结构体__


```
type A struct {
	a int
}
va := A{a:123}
vb := &A{a:123}
vc := new(A)

func NewA() (a *A) {  //这个是一个很常用的模式
    a = &A{}
    return
}
```

### Pointer

指针不多说，未初始化的指针值是nil。

```
var p *Point
var pp *[4]int
```

### Function

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

### Interface
接口类型定义了一个方法组。接口类型的变量可以存放任何实现了这个方法组的类型的值。未初始化的接口类型值为 nil。
```
// A simple File interface
interface {
	Read(b Buffer) bool
	Write(b Buffer) bool
	Close()
}
```

__特殊的接口类型__

```
interface{}
```

任何类型都实现了空接口



### Channel

管道用来实现并发编程中的通信。为初始化的管道的值是nil。

```
chan T          // can be used to send and receive values of type T
chan<- float64  // can only be used to send float64s
<-chan int      // can only be used to receive ints
```

初始化管道要用函数`make`
用内建函数 `close()` 来关闭一个管道


```
channel := make(chan int, 100)
channel <- 3 (向管道内放一个数字)
v1 := <-channel （从管道接收一个数字）
var x, ok = <-channel
close(channel)
```

__发送消息会阻塞__

Communication blocks until the send can proceed. A send on an unbuffered channel can proceed if a receiver is ready. A send on a buffered channel can proceed if there is room in the buffer. A send on a closed channel proceeds by causing a run-time panic. A send on a nil channel blocks forever.

__接收消息会阻塞__

The expression blocks until a value is available. Receiving from a nil channel blocks forever. A receive operation on a closed channel can always proceed immediately, yielding the element type's zero value after any previously sent values have been received.

__处理阻塞__

可以用 select 来处理阻塞

```
//无阻塞的接收
select {
    case msg := <-messages:
        fmt.Println("received message", msg)
    default:
        fmt.Println("no message received")
    }
    
//无阻塞的发送
select {
    case messages <- msg:
        fmt.Println("sent message", msg)
    default:
        fmt.Println("no message sent")
    }
    
//同时处理多个管道    
select {
    case msg := <-messages:
        fmt.Println("received message", msg)
    case sig := <-signals:
        fmt.Println("received signal", sig)
    default:
        fmt.Println("no activity")
    }

//超时处理    
select {
    case msg := <-messages:
        fmt.Println("received message", msg)
    case sig := <-signals:
        fmt.Println("received signal", sig)
    default time.After(time.Second * 3):
        fmt.Println("no activity")
    }
```

### 自定义类型

使用`type`来自定义类型

```
type IntArray [16]int

type (
	Point struct{ x, y float64 }
	Polar Point
)

type TreeNode struct {
	left, right *TreeNode
	value *Comparable
}

type Block interface {
	BlockSize() int
	Encrypt(src, dst []byte)
	Decrypt(src, dst []byte)
}
```

