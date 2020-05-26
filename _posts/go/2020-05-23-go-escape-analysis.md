---
layout: post
title: go内存逃逸分析
categories: 编程语言
description: 介绍go的内存逃逸分析，以及容易出现内存逃逸的场景
keywords:  逃逸分析,内存逃逸
---

> 说到内存分配，我们需要有一个常识性的认知：栈内存的分配较廉价，而堆内存分配相对昂贵。

## 一、内存结构——堆和栈

在高级编程语言中，内存分配的数据结构是堆和栈，其中栈是一种有序的数据结构，而堆则较为无序。像Java、Go、C#，一般数据类型的值存储在栈中，而引用数据类型的值存储在堆中。堆中的内存一般是需要回收的，而栈中内存在方法返回后一般也就自动回收掉了。一些非垃圾回收(GC)的语言，如C，存储在堆中的数据如果没有及时回收则容易导致内存泄露，而基于垃圾回收的语言（如Go、Java、C#）则不用太担心这个问题。

![image-20200524090334239]({{ site.url }}/assets/go/image-20200524090709709.png)

因为，基于垃圾回收的语言，GC会自动回收堆中（非栈中）不再被引用的对象，也就是说GC回收器并不管栈的数据。

程序运行时，依赖运行时栈进行函数间调用和返回，此时在内存中的数据结构称为栈帧。栈帧的作用是传递参数，存储返回信息，存储寄存器信息以及存储局部变量。

栈中的数据是不需要GC回收的，正常情况下，下图中的P函数调用Q，当Q函数正常返回后，Q的栈帧内存就被自动释放了。

![image-20200524091711515]({{ site.url }}/assets/go/image-20200524091711515.png)

## 二、Go内存逃逸
试想一下，如果程序运行时数据都分配在栈帧内存中，堆只有少量的数据，甚至是没有数据，那么GC的工作是不是就会轻松很多？对Java、Go这类语言简直太友好了！

然而,栈内存是在编译时分配的，而堆才更适合内存的动态分配（运行时分配）。很多时候，程序需要在真正运行时才知道需要多少内存，以Go为例，编译器在编译时如果发现无法确切知道需要分配多少内存给某个对象，那么就会让这个对象”逃逸”，把它分配到堆内存中，因此后面就需要GC回收器回收它。

GC意味着性能的消耗，因为回收器要停止程序的运行，并搜集和清理不再使用的堆内数据，垃圾回收算法如标记整理、标记清除等等，在Java JVM中的演进就非常多。

Go的内存逃逸相比HotSpot JVM来说更加简单，基本的原则就是，如果从声明变量的函数返回对变量的引用，那么这个变量就发生了『逃逸』。因为尽管函数返回了，但这个变量仍然被引用，所以必须将它分配在堆内存中。

![image-20200524092345596]({{ site.url }}/assets/go/image-20200524092345596.png)

#### 逃逸分析

go build 工具中可以添加参数  -gcflags '-m' 可以用来分析内存逃逸的情况，最多可以提供 4 个 "-m", m 越多则表示分析的程度越详细，一般情况下我们可以采用两个 m 分析。

> 使用这个命令也可以获得相同结果：go tool compile "-m" main.go

```shell
$ go build -gcflags '-m -l' main.go
# command-line-arguments
./main.go:7:13: x escapes to heap
./main.go:7:13: main ... argument does not escape
```

> 这里的 -l参数即 disable inline，使编译器在编译时取消函数内联优化。

有了上面的方法，下面我们来具体看下一个段Go代码以及它的逃逸分析：

```go
package escape

type S struct{}

func main() {
	var x S
	_ = identity(x)
}
func identity(x S) S {
	return x
}
```

```shell
-> % go build -gcflags '-m=1' no_escape.go       
# command-line-arguments
./no_escape.go:9:6: can inline identity
./no_escape.go:5:6: can inline main
./no_escape.go:7:14: inlining call to identity
```

可以看到，没有类似**escapes to heap**的输出，上述代码并不会发生内存逃逸，内存并不会分配到堆中，栈帧内包含了所有对象的分配，在函数调用结束后，内存自动释放。那么，究竟什么情况下，Go的内存会发生逃逸呢？

Go内存逃逸，主要有四种场景：

- 指针逃逸
- 栈空间不足逃逸（空间开辟过大）
- 动态类型逃逸（不确定大小）
- 闭包引用对象逃逸

以上场景过于抽象，但确是普适性的场景，实际使用上，其实可以分为如下典型场景：

- 发送指针或带有指针的值到 channel 中。在编译时，是没有办法知道哪个 goroutine 会在 channel 上接收数据。所以编译器没法知道变量什么时候才会被释放。

-  在一个切片上存储指针或带指针的值。一个典型的例子就是 []*string。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。

-  slice 的背后数组被重新分配了，因为 append 时可能会超出其容量(cap)。slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。

-  在 interface 类型上调用方法。在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r, 调用 r.Read(b) 会使得 r 的值和切片 b 的背后存储都逃逸掉，所以会在堆上分配。

##### 指针逃逸

案例一：

```go
package escape

type Q struct {
	h *int
}

func method() {
	aChan := make(chan int)
	aChan <- 0
	bChan := make(chan Q)
	s := new(Q)
	value := 1
	s.h = &value
	bChan <- *s
}

func main1() {
	method()
}

```

```shell
-> % go build -gcflags '-m=1' pointer_in_chan.go 
# command-line-arguments
./pointer_in_chan.go:7:6: can inline method
./pointer_in_chan.go:17:6: can inline main1
./pointer_in_chan.go:18:8: inlining call to method
./pointer_in_chan.go:13:8: &value escapes to heap
./pointer_in_chan.go:12:2: moved to heap: value
./pointer_in_chan.go:8:15: method make(chan int) does not escape
./pointer_in_chan.go:10:15: method make(chan Q) does not escape
./pointer_in_chan.go:11:10: method new(Q) does not escape
./pointer_in_chan.go:18:8: &value escapes to heap
./pointer_in_chan.go:18:8: moved to heap: value
./pointer_in_chan.go:18:8: main1 make(chan int) does not escape
./pointer_in_chan.go:18:8: main1 make(chan Q) does not escape
./pointer_in_chan.go:18:8: main1 new(Q) does not escape
```

案例二：

```go
package main

type S struct {}

func main() {
  var x S
  _ = *ref(x)
}

func ref(z S) *S {
  return &z
}
```

```shell
-> % go build -gcflags '-m=1' pointer_escape.go 
# command-line-arguments
./pointer_escape.go:10:6: can inline ref
./pointer_escape.go:5:6: can inline main
./pointer_escape.go:7:10: inlining call to ref
./pointer_escape.go:7:10: main &z does not escape
./pointer_escape.go:11:9: &z escapes to heap
./pointer_escape.go:10:10: moved to heap: z
```

##### 栈空间不足逃逸

案例一：

```go
package escape

import "math/rand"

func allo1() {
	s := make([]byte, 1, 1*1024)
	_ = s
}

func allo2() {
	s := make([]byte, 1, 63*1024) // 63k
	_ = s
}

func allo3() {
	s := make([]byte, 1, 64*1024) // 64k
	_ = s
}

func allo4() {
	randSize := rand.Int()
	s := make([]*string, 0, randSize)
	str := "hello"
	s = append(s, &str)
	_ = s
}

func allo5() {
	s := make([]*string, 0, 5)
	str := "hello"
	s = append(s, &str)
	_ = s
}
```

```shell
-> % go build -gcflags '-m=2' not_enough_space.go
# command-line-arguments
./not_enough_space.go:5:6: can inline allo1 as: func() { s := make([]byte, 1, 1 * 1024); _ = s }
./not_enough_space.go:10:6: can inline allo2 as: func() { s := make([]byte, 1, 63 * 1024); _ = s }
./not_enough_space.go:15:6: can inline allo3 as: func() { s := make([]byte, 1, 64 * 1024); _ = s }
./not_enough_space.go:20:6: cannot inline allo4: function too complex: cost 84 exceeds budget 80
./not_enough_space.go:28:6: can inline allo5 as: func() { s := make([]*string, 0, 5); str := "hello"; s = append(s, &str); _ = s }
./not_enough_space.go:6:11: allo1 make([]byte, 1, 1 * 1024) does not escape
./not_enough_space.go:11:11: allo2 make([]byte, 1, 63 * 1024) does not escape
./not_enough_space.go:16:11: make([]byte, 1, 64 * 1024) escapes to heap
./not_enough_space.go:16:11:    from make([]byte, 1, 64 * 1024) (too large for stack) at ./not_enough_space.go:16:11
./not_enough_space.go:22:11: make([]*string, 0, randSize) escapes to heap
./not_enough_space.go:22:11:    from make([]*string, 0, randSize) (non-constant size) at ./not_enough_space.go:22:11
./not_enough_space.go:24:16: &str escapes to heap
./not_enough_space.go:24:16:    from append(s, &str) (appended to slice) at ./not_enough_space.go:24:12
./not_enough_space.go:23:2: moved to heap: str
./not_enough_space.go:31:16: &str escapes to heap
./not_enough_space.go:31:16:    from append(s, &str) (appended to slice) at ./not_enough_space.go:31:12
./not_enough_space.go:30:2: moved to heap: str
./not_enough_space.go:29:11: allo5 make([]*string, 0, 5) does not escape
```



##### 动态类型逃逸（不确定大小）

案例一：

```go
package escape

import "math"

type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}

type Shape interface {
	Area() float64
}

func getArea(shape Shape) {
	shape.Area()
}

func main2() {
	rectangle := Rectangle{12, 6}
	circle := Circle{10}
	rectangle.Area()
	circle.Area()

	getArea(rectangle)
}

```

```shell
-> % go build -gcflags '-m=1' interface_method.go 
# command-line-arguments
./interface_method.go:10:6: can inline Rectangle.Area
./interface_method.go:18:6: can inline Circle.Area
./interface_method.go:26:6: can inline getArea
./interface_method.go:33:16: inlining call to Rectangle.Area
./interface_method.go:34:13: inlining call to Circle.Area
./interface_method.go:36:9: inlining call to getArea
./interface_method.go:26:14: leaking param: shape
./interface_method.go:36:9: rectangle escapes to heap
<autogenerated>:1: inlining call to Circle.Area
<autogenerated>:1: (*Circle).Area .this does not escape
<autogenerated>:1: inlining call to Rectangle.Area
<autogenerated>:1: (*Rectangle).Area .this does not escape
<autogenerated>:1: leaking param: .this
```

##### 闭包引用对象逃逸

```go
package escape

func closure() {
	var y int
	func(p *int, x int) {
		*p = x
	}(&y, 42)

	x := 0 // BAD: x escapes
	defer func(p *int) {
		*p = 1
	}(&x)

}

func main4() {
	closure()
}
```

```shell
-> % go build -gcflags '-m=1' closure_eacape.go  
# command-line-arguments
./closure_eacape.go:5:2: can inline closure.func1
./closure_eacape.go:7:3: inlining call to closure.func1
./closure_eacape.go:10:8: can inline closure.func2
./closure_eacape.go:16:6: can inline main4
./closure_eacape.go:12:4: &x escapes to heap
./closure_eacape.go:9:2: moved to heap: x
./closure_eacape.go:7:4: closure &y does not escape
./closure_eacape.go:10:8: closure func literal does not escape
./closure_eacape.go:10:13: closure.func2 p does not escape
```

##  三、如何避免内存逃逸
Go内存逃逸分析有助于我们写出性能较好的程序，除了掌握分析技巧，还需要了解一些常用的技巧，这样平时在写代码的时候就能尽量地避免写出容易发生内存逃逸的程序：

- 尽量不要传递引用或者返回引用
- 减少使用可变大小对象，如slice或map
- make初始化时尽量指定常量大小
- 在调用频繁的地方慎用 interface

本文简单介绍了内存逃逸分析，讲解了如何使用Go语言内置的内存逃逸分析工具，由于篇幅原因，没有介绍优化前后的程序性能对比。性能对比，也许才更有体感，这个我想后面再来分析~