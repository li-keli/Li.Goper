---
title: "Go_No2"
date: 2020-03-01T09:29:11+08:00
draft: false
tags: [Go, 刷题]
featured_image: "https://oss.likeli.top/uPic/20200401094728"
categories: 刷题
---

# go 试卷 NO.2

### 1.是否可以编译通过？如果通过，输出什么？

```go
func main() {
	i := GetValue()

	switch i.(type) {
	case int:
		println("int")
	case string:
		println("string")
	case interface{}:
		println("interface")
	default:
		println("unknown")
	}

}

func GetValue() int {
	return 1
}
```

#### 答案

编译失败，type只能使用在interface上，异常如下图

![](https://oss.likeli.top/uPic/20200401094044)



### 2.下面函数有什么问题？

```go
func funcMui(x, y int) (sum int, error) {
	return x + y, nil
}
```

#### 答案

返回参数定义异常

#### 补充

返回参数的定义：

* 在函数有多个返回值时，只要有一个返回值有指定命名，其他的也必须有命名。
* 如果返回值有多个，返回值必须加上括号
* 如果只有一个返回值并且有命名，也需要加上括号



### 3.是否可以编译通过？如果通过，输出什么？

```go
package main

func main() {

	println(DeferFunc1(1))
	println(DeferFunc2(1))
	println(DeferFunc3(1))
}

func DeferFunc1(i int) (t int) {
	t = i
	defer func() {
		t += 3
	}()
	return t
}

func DeferFunc2(i int) int {
	t := i
	defer func() {
		t += 3
	}()
	return t
}

func DeferFunc3(i int) (t int) {
	defer func() {
		t += i
	}()
	return 2
}
```

#### 答案

<details>
  4,1,3
</details>

#### 补充

此题考查defer和函数返回值。

* defer会在函数结束前执行
* 函数返回值名字会在函数起始处被初始化为对应类型的零值，并且作用域为整个函数



### 4.是否可以编译通过？如果通过，输出什么？

```go
func main() {
	list := new([]int)
	list = append(list, 1)
	fmt.Println(list)
}
```

#### 答案

编译错误，如图：

![20200304162306](https://oss.likeli.top/uPic/20200401094045)

new只分配内存，并不初始化内存，只是将其置零。

可改为如下操作：

```go
func Test_demo9(t *testing.T) {
   list := make([]int, 1)
   list = append(list, 1)
   fmt.Println(list)
}
```



### 5.是否可以编译通过？如果通过，输出什么？

```go
package main

import "fmt"

func main() {
	s1 := []int{1, 2, 3}
	s2 := []int{4, 5}
	s1 = append(s1, s2)
	fmt.Println(s1)
}
```

#### 答案

编译错误，语法糖。append切片的时候加上`...`

正确写法：`s1 = append(s1, s2...)`



### 6.是否可以编译通过？如果通过，输出什么？

```go
func main() {
	sn1 := struct {
		age  int
		name string
	}{age: 11, name: "qq"}
	sn2 := struct {
		age  int
		name string
	}{age: 11, name: "qq"}

	if sn1 == sn2 {
		fmt.Println("sn1 == sn2")
	}

	sm1 := struct {
		age int
		m   map[string]string
	}{age: 11, m: map[string]string{"a": "1"}}
	sm2 := struct {
		age int
		m   map[string]string
	}{age: 11, m: map[string]string{"a": "1"}}

	if sm1 == sm2 {
		fmt.Println("sm1 == sm2")
	}
}
```

#### 答案

进行结构比较的时候，只有相同类型的结构体才可以比较，结构体是否相同不但**与属性类型个数相关，还与属性顺序相关**。

另外若是结构体相同，但是结构体属性中有不可以比较的类型，如：**map, slice**。如果该结构体属性都可以比价，那么就可以用"=="来进行比较。

体重sm1和sm2可以使用`reflect.DeepEqual`进行比较

```go
if reflect.DeepEqual(sm1, sm2) {
	fmt.Println("sm1 == sm2")
} else {
	fmt.Println("sm1 != sm2")
}
```



### 7.是否可以编译通过？如果通过，输出什么？

```go
func Foo(x interface{}) {
	if x == nil {
		fmt.Println("empty interface")
		return
	}
	fmt.Println("non-empty interface")
}
func main() {
	var x *int = nil
	Foo(x)
}
```

#### 答案

`non-empty interface`

此题考查interface内部结构，空的interface内部实现和带有方法集的interface内部实现是不一样的



### 8.是否可以编译通过？如果通过，输出什么？

```go
func GetValue(m map[int]string, id int) (string, bool) {
	if _, exist := m[id]; exist {
		return "存在数据", true
	}
	return nil, false
}
func main() {
	intmap := map[int]string{
		1: "a",
		2: "bb",
		3: "ccc",
	}

	v, err := GetValue(intmap, 3)
	fmt.Println(v, err)
}
```

#### 答案

编译错误 ，`Cannot use 'nil' as type string in return argument `

函数返回值类型nil可以用作interface、function、pointer、map、slice和channel的“空值”。但是如果不特别指定的话，Go语言不能识别类型，所以会报错。



### 9.是否可以编译通过？如果通过，输出什么？

```go
const (
	x = iota
	y
	z = "zz"
	k
	p = iota
)

func main() {
	fmt.Println(x, y, z, k, p)
}
```

#### 答案

<details>
  0 1 zz zz 4
</details>