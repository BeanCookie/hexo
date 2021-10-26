---
title: Golang基础系列之变量、指针与基本数据结构
date: 2020-09-09 13:54:00
categories:
- Golang
---

#### 变量

**Golang**身为一种强类型语言，却拥有强大的类型推断系统。编程语言的究竟属于强类型还是弱类型并没有权威的定义，我们很多时候也只能根据现象和特性从直觉上进行判断：强类型的编程语言在编译期间会有着更严格的类型限制，也就是编译器会在编译期间发现变量赋值、返回值和函数调用时的类型错误，而弱类型的语言在出现类型错误时可能会在运行时进行隐式的类型转换，在类型转换时可能会造成运行错误。

[Golang网页编辑器](https://golang.org/)

```go
package main

import "fmt"

func main() {
	var a = 1
	fmt.Printf("a Type is %T\n", a)
	// a = 1.1
	a = 1.00
	var b = 2.0
	fmt.Printf("b Type is %T\n", b)
	c := "three"
	fmt.Printf("c Type is %T\n", c)
}
```

**Golang**是如何实现类型推断的呢？



#### 指针

**Golang**作为一门新生语言，并没有沿用面向对象的主流思想，而是返璞归真的引入了指针。**Golang**中的指针并不是**C**中BUG般的存在，而是加以了诸多限制让它看起来更像Java中的引用。

```go
package main

import "fmt"

type MyStruct struct {
	i int
}

func add(a *MyStruct) {
	fmt.Printf("add:\t %v %p\n", a, &a)
}

func main() {
	a := 1
	pA := &a
	// pA++
	fmt.Printf("%v %p\n", a, &pA)

	var b = &MyStruct{i: 1}
	fmt.Printf("main:\t %v %p\n", b, &b)
	add(b)
	fmt.Printf("main:\t %v %p\n", b, &b)
}
```

