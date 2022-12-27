# Golang经典“坑”之for range

## 问题描述

之前写一段代码如下:

```go
package main

import (
    "fmt"
	"time"
)

func testClosuer() {
	ss := []string{"bb", "aa"}
	for idx, x := range ss {
		go func(i int) {
			fmt.Printf("%v: %v\n", i, x)
		}(idx)
	}
}

func main() {
  testClosuer()
}
```

满心以为结果会是:

```txt
0: bb
1: aa
```

或者:

```txt
1: aa
0: bb
```

然而实际运行出来的结果却是:(两行结果位置可能对调)

```txt
1: aa
0: aa
```

**也就是说，不管运行多少次，goroutine中获取的x的值，都是ss这个string slice中的第二个值。**

## 分析

这显然和我们直觉的理解不同。直觉的理解，此段程序应该是起了两个goroutine，两个goroutine分别打印ss这个string slice的两个元素的内容以及所在的位置。但实际上，**goroutine正确打印了自己负责元素的位置，而没有正确打印负责元素的内容。那么，区别在哪里？**

### 1. goroutine中i和x的区别

回顾一下产生问题的test函数：

```go
func testClosuer() {
	ss := []string{"bb", "aa"}
	for idx, x := range ss {
		go func(i int) {
			fmt.Printf("%v: %v\n", i, x)
		}(idx)
	}
}
```

代码块中，匿名goroutine函数打印时，使用了两个变量，其中i为参数变量，x为闭包访问for语句中定义的x变量。区别在于：
* i的生命周期跟随所在goroutine的产生和消亡，i的值，则由外层for语句定义的idx变量的值复制而来
* x的声明周期跟随goroutine外层for循环语句产生和消亡，x的值就是外层for语句定义的x变量的值

从两者的区别来看，我们目前只能怀疑问题出在闭包上。

### 2. golang闭包的实现原理
个人一直以为闭包是个很神奇的东西，能够让内层函数直接访问外层的变量。既然怀疑问题出在闭包，那么就得从闭包的实现开始寻找思路。

闭包的实现原理，<<Go语言学习笔记>>中有解释(P91)，goroutine能够使用x变量，是由于函数创建时，函数拿到了x的**地址**。

想到了什么？

Yes。两个goroutine获取的是同一个x变量的地址，并且在访问时，x的值已经因为for range的循环而被改变。

验证一下：

```go
func testClosuer() {
	ss := []string{"bb", "aa"}
	for idx, x := range ss {
		go func(i int) {
			fmt.Printf("%v: %v, %v\n", i, &x, x)
		}(idx)
	}

	time.Sleep(time.Minute)
}
```
结果:

```txt
0: 0xc420011200, aa
1: 0xc420011200, aa
```

确实如此。

由此，问题原因找到了：**两者访问的是同一个地址，同时由于goroutine的创建需要时间，因此首次循环产生的goroutine到访问x时，其内容已经改变**。

这里就涉及到另外一个问题：golang中range每次迭代，对于变量，是重新定义后初始化，还是变量定义后仅赋值？

### 3.golang中range的原理
golang中，类似a, b := range xx的语句，a,b均仅在第一次循环时定义，也即，range每次循环，仅改变a,b的值而非重新定义一个变量再赋值。
验证一下：

```go
func testClosuer() {
	ss := []string{"bb", "aa"}
	for idx, x := range ss {
		fmt.Printf("idx: %v, x: %v\n", &idx, &x)
		go func(i int) {
			fmt.Printf("%v: %v, %v\n", i, &x, x)
		}(idx)
	}

	time.Sleep(time.Minute)
}
```

```txt
idx: 0xc420016748, x: 0xc420011200
idx: 0xc420016748, x: 0xc420011200
0: 0xc420011200, aa
1: 0xc420011200, aa
```
确实如此。

## 总结
问题的产生，是由于range每次迭代均使用同一个变量，而goroutine访问闭包变量，也是通过地址来进行访问，再加上goroutine创建时的时间消耗，导致两个goroutine实际运行时，range以及迭代完成，因此访问到的闭包变量为range最后一次迭代的值。

要解决也简单，可用方法很多，比如：
* 更改goroutine使用变量的方式，不使用闭包，而使用参数，来保证值传递。
* goroutine中，直接定义临时变量并初始化为x的值

#CS/Go 