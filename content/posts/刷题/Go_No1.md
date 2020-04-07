---
title: "Go_No1"
date: 2020-03-01T09:29:11+08:00
draft: false
tags: [Go, 面试题]
featured_image: "https://oss.likeli.top/uPic/20200401094728"
categories: 刷题
---

# go 试卷NO.1

### 1.写出下面代码输出的结果

```go
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() { fmt.Println("打印前") }()
	defer func() { fmt.Println("打印中") }()
	defer func() { fmt.Println("打印后") }()

	panic("触发异常")
}
```

#### 答案

<details>
  === RUN   Test_demo0
打印后
打印中
打印前
--- FAIL: Test_demo0 (0.00s)
panic: 触发异常 [recovered]
	panic: 触发异常
</details>

#### 解析

`defer` 先进后出，逆序执行。而`panic` 可能在任何位



### 2.以下代码有什么问题

```go
type student struct {
	Name string
	Age  int
}

func pase_student() {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}
}
```

#### 答案

<details>
map[string]*notify_center.student{"qian":(*notify_center.student)(0xc0000d6000), "sun":(*notify_center.student)(0xc0000d6000), "zhao":(*notify_center.student)(0xc0000d6000)}
</details>

#### 解析

在**range**循环中**stu**变量会利用同一块内存空间来收集集合中的一个值，

因此代码中的**stu**值会变化，但是其指向的内存地址都是同一个。



### 3.下面代码会输出什么，并说明原因

```go
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("Ai: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("Bi: ", i)
			wg.Done
}
```

#### 答案

<details>
=== RUN   Test_demo2
Bi:  9
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Ai:  10
Bi:  0
Bi:  1
Bi:  2
Bi:  3
Bi:  4
Bi:  5
Bi:  6
Bi:  7
Bi:  8
--- PASS: Test_demo2 (0.00s)
PASS
</details>

#### 解析

`sync.WaitGroup`可以一直等到所有goroutine全部执行完毕，并阻塞执主线程。

不过需要注意的是，他们的执行结果是没有顺序的，调度器不能保证多个goroutine执行的顺序，且退出时不会等待他们结束。

#### 补充

WaitGroup共有三个方法：Add，Done，Wait。

内容简单，这里不过多做描述，具体看[官方API](https://golang.google.cn/pkg/sync/#WaitGroup)



### 4.下面的代码会输出什么

```go
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```

#### 答案

<details>
showA
showB
</details>

#### 解析

首先，Go中没有继承关系，叫做`组合（composite）`，而`t.ShowA()`实则调用的`t.people.ShowA()`，也就是说在组合的struct里面，go会优先调用本身的方法，如果本身没有此方法，就会去调用其组合结构的方法。

#### 补充

若是组合的结构里面有相同的方法，编译器会报错：

![image-20200302125219902](https://oss.likeli.top/PicGo-img/20200304124737.png?x-oss-process=style/smail_img_likeli)



### 5.下面的代码会触发异常吗？请详细说明

```go
func main() {
	runtime.GOMAXPROCS(1)
	int_chan := make(chan int, 1)
	string_chan := make(chan string, 1)
	int_chan <- 1
	string_chan <- "hello"
	select {
	case value := <-int_chan:
		fmt.Println(value)
	case value := <-string_chan:
		panic(value)
	}
}
```

#### 答案

<details>
  代码有可能会触发异常，也可能不会
</details>

#### 解析

**select随机性**，select会随机选择一个可用通道做收发操作。所以代码有可能出发异常，也有可能不会。单个chan如果无缓冲时，将会阻塞。但结合select可以在多个chan间等待执行，有三个原则：

* select中只要有一个case能return，则立刻return
* 当如果同一时间有多个case均能return则伪随机方式抽取任意一个执行
* 如果没有一个case能return则执行default



### 6.下面代码输出什么？

```go
func calc(index string, a, b int) int {
    ret := a + b
    fmt.Println(index, a, b, ret)
    return ret
}

func main() {
    a := 1
    b := 2
    defer calc("1", a, calc("10", a, b))
    a = 0
    defer calc("2", a, calc("20", a, b))
    b = 1
}
```

#### 答案

<details>
10 1 2 3
20 0 2 2
2  0 2 2
1  1 3 4
</details>

#### 解析

* defer先进后出，逆序执行；
* defer会优先执行内部的方法，如上面的`calc("10", a, b)`

#### 补充

补充一点关于defer的闭包，若将执行方法改造如下：

```go
  a := 1
	b := 2
	defer calc("1", b, calc("10", a, b))
	a = 0
	defer func() {
		calc("2", b, calc("20", a, b))
	}()
	b = 1
```

第二个defer执行了一个闭包函数，也就是在函数内的b和外面的b是同一个变量，内存地址也是一样的。所以末尾的b=1赋值对此闭包函数是生效的。

那么对应的答案是：

<details>
10 1 2 3
20 0 1 1
2  1 1 2
1  2 3 5
</details>


### 7.请写出以下输入内容

```go
func main() {
    s := make([]int, 3)
    s = append(s, 1, 2, 3)
    fmt.Println(s)
}
```

#### 答案

<details>
[0 0 0 1 2 3]
</details>

#### 解析

make初始化的数组是有默认值的，此处的int数组默认值为0



### 8.下面的代码有什么问题？

```go
type UserAges struct {
	ages map[string]int
	sync.Mutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}
```

#### 答案

Get方法没有实现锁，可能会出现`fatal error: concurrent map read and map write`

#### 补充

```go
func (ua *UserAges) Get(name string) int {
	ua.Lock()
	defer ua.Unlock()
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}
```



### 9.下面代码迭代会有什么问题？

```go
type threadSafeSet struct {
	sync.RWMutex
	s []interface{}
}

func (set *threadSafeSet) Iter() <-chan interface{} {
	ch := make(chan interface{})
	go func() {
		set.RLock()

		for elem := range set.s {
			ch <- elem
		}

		close(ch)
		set.RUnlock()

	}()
	return ch
}
```

#### 答案

题目中要求看迭代部分，那么，此题中的chan是无缓冲的，所以不能全部迭代，若要完成全部迭代，则要改为有缓冲的channel，如下：

```go
type threadSafeSet struct {
	sync.RWMutex // 读写锁
	s []interface{}
}

func (set *threadSafeSet) Iter() <-chan interface{} {
	// 增加缓冲
	ch := make(chan interface{}, len(set.s))
	go func() {
		set.RLock()

		for elem, value := range set.s {
			ch <- value
			fmt.Printf("Iter: %d %v \n", elem, value)
		}

		close(ch)
		set.RUnlock()
	}()
	return ch
}

func Test_demo7(t *testing.T) {
	th := threadSafeSet{
		s: []interface{}{"1", "2"},
	}
	for b := range th.Iter() {
		fmt.Println("b ===", b)
	}
}
```

> 相关 [Go中的sync.Mutex和sync.RWMutex](https://www.jianshu.com/p/679041bdaa39)



### 10.以下代码能编译过去吗？为什么？

```go
package main

import (
	"fmt"
)

type People interface {
	Speak(string) string
}

type Student struct{}

func (stu *Student) Speak(think string) (talk string) {
	if think == "bitch" {
		talk = "You are a good boy"
	} else {
		talk = "hi"
	}
	return
}

func main() {
	var peo People = Student{}
	think := "bitch"
	fmt.Println(peo.Speak(think))
}
```

#### 答案

编译错误，`Type does not implement 'People' as 'Speak' method has a pointer receiver `

一个接口包含两层意思：

* 它是一个方法的集合
* 它是一个类型

> Go中所有的东西都是按值传递的。每次调用函数时，传入的数据就会被复制。对于具有值接受者的方法，在调用该方法时将复制该值。

所以上面的Student类型的值可能会有很多 *Student 类型的指针指向它，如果我们尝试通过Student类型的值来调用 *Student 的方法，根本就不知道对应的是哪个指针。相反，如果Student类型上有一个方法，通过 *Student 来调用这个方法可以确切的找到该指针对应的Student类型的值，从而调用上面的方法。

> 此处参考 [如何在Go中使用接口](https://www.jianshu.com/p/88c4ed564aa9)



### 11.以下代码打印出来的什么内容，说出为什么

```go
package main

import (
	"fmt"
)

type People interface {
	Show()
}

type Student struct{}

func (stu *Student) Show() {

}

func live() People {
	var stu *Student
	return stu
}

func main() {
	if live() == nil {
		fmt.Println("AAAAAAA")
	} else {
		fmt.Println("BBBBBBB")
	}
}
```

#### 答案

<details>
  BBBBBBB
</details>

此题考查interface内部结构。

go中的接口分为两种，一种是空的接口类似这样的：

```go
var in interface{}
```

另一种如题目的：

```go
type People interface{
  Show()
}
```

这两种接口的底层结构是不一样的，如下：

```go
type eface struct { //空接口
	_type *_type         //类型信息
	data  unsafe.Pointer //指向数据的指针(go语言中特殊的指针类型unsafe.Pointer类似于c语言中的void*)
}
type iface struct { //带有方法的接口
	tab  *itab          //存储type信息还有结构实现方法的集合
	data unsafe.Pointer //指向数据的指针(go语言中特殊的指针类型unsafe.Pointer类似于c语言中的void*)
}
type _type struct {
	size       uintptr //类型大小
	ptrdata    uintptr //前缀持有所有指针的内存大小
	hash       uint32  //数据hash值
	tflag      tflag
	align      uint8    //对齐
	fieldalign uint8    //嵌入结构体时的对齐
	kind       uint8    //kind 有些枚举值kind等于0是无效的
	alg        *typeAlg //函数指针数组，类型实现的所有方法
	gcdata     *byte
	str        nameOff
	ptrToThis  typeOff
}
type itab struct {
	inter  *interfacetype //接口类型
	_type  *_type         //结构类型
	link   *itab
	bad    int32
	inhash int32
	fun    [1]uintptr //可变大小 方法集合
}
```

可以从上看出iface比eface中间多出了一层itab结构。itab存储_type信息和[]fun方法集，从上面的结构我们就可以得出，因为data指向看nil并不代表interface是nil，**所以返回值并不为空**，这里的fun(方法集)定义了接口的接收规则，在编译的过程中需要验证是否实现接口结果。