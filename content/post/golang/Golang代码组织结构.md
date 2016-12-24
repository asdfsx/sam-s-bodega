+++
date = "2016-12-11T00:03:11+08:00"
title = "Golang代码组织结构"
tags = [
  "golang"
]
draft = false
topics = [
  "topic 1",
]
keywords = [
  "golang"
]
description = "Golang代码组织结构"
author = "asdfsx"
type = "post"

+++

结合官方的文档，还有《The way to go》先来熟悉一下golang的代码结构。以下来自官方文档

```
Go programs are constructed by linking together packages. A package in turn is constructed from one or more source files that together declare constants, types, variables and functions belonging to the package and which are accessible in all files of the same package. Those elements may be exported and used in another package.

Each source file consists of a package clause defining the package to which it belongs, followed by a possibly empty set of import declarations that declare packages whose contents it wishes to use, followed by a possibly empty set of declarations of functions, types, variables, and constants.

A package clause begins each source file and defines the package to which the file belongs.

A set of files sharing the same PackageName form the implementation of a package. An implementation may require that all source files for a package inhabit the same directory.

Within a package, package-level variables are initialized in declaration order but after any of the variables they depend on.

Variables may also be initialized using functions named init declared in the package block, with no arguments and no result parameters.

A complete program is created by linking a single, unimported package called the main package with all the packages it imports, transitively. The main package must have package name main and declare a function main that takes no arguments and returns no value.

Program execution begins by initializing the main package and then invoking the function main. When that function invocation returns, the program exits. It does not wait for other (non-main) goroutines to complete.

......
```

简单概括一下

- golang的项目有多个包组成  
- 每个包由多个文件组成  
- 每个文件包含几部分：必须有的package声明，可能存在的import声明，可能存在的常量、类型、全局变量、函数等的声明
- 同一个包内的文件使用同一个package名称
- 同一个包内的文件要在同一个目录下
- 在包中，包级别的变量会按照声明顺序初始化
- 变量也可以由init函数来初始化，init含税可能会有多个
- 整个程序是由`main`包创建的，这个包必须包含一个叫做`main`的函数
- 程序执行是，先初始化`main`包，然后执行`main`函数

如果要再简单直白一点

- 一个包就是一个目录，包名就是目录名（其实一个目录下是可以有多个包存在的，用前边这种方式可以让代码结构简单明了）
- 目录下存放该包里的所有文件
- 每个包中可以有多个init函数，该函数会在import包时被自动调用
- 整个程序的入口为一个`main`函数，该函数没有参数，没有返回值，属于一个不被引用的包`main`包。

也可以用一张图来概括
![](http://ohrdj7osp.bkt.clouddn.com/20150416173122272.png)


补充
----
* init函数的一个常见用途是包的初始化和注册  
举例：mozilla的heka项目，有多个插件组成，每个插件都是一个独立的包，插件的初始化和注册都是由包中的init函数来实现的。

作为一个python程序员，有没有想到那个`__init__.py`?不过这个init函数不是必须的。

* 同一个目录里如果有多个main函数，不能简单的使用`go build`来编译，会提示一个main函数被重复定义，只能一个一个文件来build。

* 抄个例子：

```
package main

import "fmt"

// Send the sequence 2, 3, 4, … to channel 'ch'.
func generate(ch chan<- int) {
	for i := 2; ; i++ {
		ch <- i  // Send 'i' to channel 'ch'.
	}
}

// Copy the values from channel 'src' to channel 'dst',
// removing those divisible by 'prime'.
func filter(src <-chan int, dst chan<- int, prime int) {
	for i := range src {  // Loop over values received from 'src'.
		if i%prime != 0 {
			dst <- i  // Send 'i' to channel 'dst'.
		}
	}
}

// The prime sieve: Daisy-chain filter processes together.
func sieve() {
	ch := make(chan int)  // Create a new channel.
	go generate(ch)       // Start generate() as a subprocess.
	for {
		prime := <-ch
		fmt.Print(prime, "\n")
		ch1 := make(chan int)
		go filter(ch, ch1, prime)
		ch = ch1
	}
}

func main() {
	sieve()
}
```