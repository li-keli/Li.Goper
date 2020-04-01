---
title: "Go_No3"
date: 2020-03-01T09:29:11+08:00
draft: false
tags: [Go, 刷题]
featured_image: "https://oss.likeli.top/uPic/20200401094728"
categories: 刷题
---

##  go 试卷 NO.3

### 1.执行下面代码会出现什么？

```go
package main
var(
    size := 1024
    max_size = size*2
)
func main()  {
    println(size, max_size)
}
```

#### 答案

语法问题，变量简短模式。

变量简短模式限制：

* 定义变量同时显示初始化
* 不能提供数据类型
* 只能在函数内部使用



### 2.下面函数有什么问题？

```go
const cl = 100
var bl = 123

func main() {
   println(&bl, bl)
   println(&cl, cl)
}
```

#### 答案

`Cannot take the address of 'cl' `

常量通常在编译器预处理阶段直接展开，做为指令数据使用



### 3.执行下面的代码会出现什么？

```go
func main() {
	for i := 0; i < 10; i++ {
	loop:
		println(i)
	}
	goto loop
}
```

#### 答案

goto不能跳转到其他函数或者内层代码

[DOC](https://golang.org/ref/spec)



### 4.编译执行下面代码会出现什么？

```go
func main() {
   type MyInt1 int
   type MyInt2 = int
   var i int = 9
   var i1 MyInt1 = i
   var i2 MyInt2 = i
   fmt.Println(i1, i2)
}
```

#### 答案

在`1.9`版本新增的`Type Alias`特性。

基于一个类型创建一个新类型，称之为`defintion`；基于一个类型创建一个别名，称之为alias。

MyInt1就是`defintion`，虽然底层类型为int类型，但是不能直接赋值，需要强转：`MyInt1(i)`；

MyInt2就是`alias`，可以直接赋值。



### 5.编译执行下面代码会出现什么？

```go
type User struct {
}
type MyUser1 User
type MyUser2 = User

func (i MyUser1) m1() {
	fmt.Println("MyUser1.m1")
}
func (i User) m2() {
	fmt.Println("User.m2")
}

func Test_demo15(t *testing.T) {
	var i1 MyUser1
	var i2 MyUser2
	i1.m1()
	i2.m2()
}
```

#### 答案

<details>
=== RUN   Test_demo15
MyUser1.m1
User.m2
--- PASS: Test_demo15 (0.00s)
</details>


考查1.9中的特性"Type Alias"

题中MyUser2完全等价于User，所以User中的方法MyUser2都有，并且其中任意一方新增了方法，另一个也会有；

但是`i1.m2()`是不能执行的，因为MyUser1没有定义该方法



### 6.编译执行下面代码会出现什么？

```go
type T1 struct {
}

func (t T1) m1() {
   fmt.Println("T1.m1")
}

type T2 = T1
type MyStruct struct {
   T1
   T2
}

func Test_demo16(t *testing.T) {
   my := MyStruct{}
   my.m1()
}
```

#### 答案

错误 `ambiguous selector my.m1`

考查1.9中的特性"Type Alias"

结果不限于方法，字段也是一样；也不限于Type Alias、Type defintion也是一样，只要有重复的方法、字段，就会有这种提示，因为不知道选择哪个好。

可如下修改：

```go
func Test_demo16(t *testing.T) {
	my := MyStruct{}
	//my.m1()
	my.T1.m1()
	my.T2.m1()
}
```



### 7.编译执行下面代码会出现什么？

```go
var ErrDidNotWork = errors.New("did not work")

func DoTheThing(reallyDoIt bool) (err error) {
   if reallyDoIt {
      result, err := tryTheThing()
      if err != nil || result != "it worked" {
         err = ErrDidNotWork
      }
   }
   return err
}

func tryTheThing() (string, error) {
   return "", ErrDidNotWork
}

func Test_demo17(t *testing.T) {
   fmt.Println(DoTheThing(true))
   fmt.Println(DoTheThing(false))
}
```

#### 答案

<details>
=== RUN   Test_demo17
<nil>
<nil>
--- PASS: Test_demo17 (0.00s)
</details>


考查**变量作用域**

因为第4行if语句块会遮罩函数作用域内的err变量，改为：

```go
func DoTheThing(reallyDoIt bool) (err error) {
   var result string
   if reallyDoIt {
      result, err = tryTheThing()
      if err != nil || result != "it worked" {
         err = ErrDidNotWork
      }
   }
   return err
}
```



### 8.执行下面代码会出现什么？

```go
func test() []func() {
   var funs []func()
   for i := 0; i < 2; i++ {
      funs = append(funs, func() {
         println(&i, i)
      })
   }
   return funs
}

func Test_demo18(t *testing.T) {
   funs := test()
   for _, f := range funs {
      f()
   }
}
```

#### 答案

<details>
=== RUN   Test_demo18
0xc00011e1b8 2
0xc00011e1b8 2
--- PASS: Test_demo18 (0.00s)
</details>


考查 `闭包延迟求值`

for循环复用局部变量 **i**，每次放入匿名函数的都是同一个变量，所以 i 都是最后一次循环的值：2

若想不一样，在for循环块内，声明一个变量储存i，然后将此变量放入匿名函数即可，如下：

```go
func test() []func() {
   var funs []func()
   for i := 0; i < 2; i++ {
      x := i
      funs = append(funs, func() {
         println(&x, x)
      })
   }
   return funs
}
```



### 9.执行下面的代码会出现什么？

```go
func test1(x int) (func(), func()) {
   return func() {
         println(x)
         x += 10
      }, func() {
         println(x)
      }
}

func Test_demo19(t *testing.T) {
   a, b := test1(100)
   a()
   b()
}
```

#### 答案

<details>
=== RUN   Test_demo19
100
110
--- PASS: Test_demo19 (0.00s)
</details>

考查：**闭包的记忆效应** 

闭包对它作用域上部的变量可以进行修改，修改引用的变量会对变量进行实际修改，变量会跟随闭包生命周期一直存在

[相关文档](http://c.biancheng.net/view/59.html)



### 10.编译执行下面代码会出现什么？

```go
func Test_demo20(t *testing.T) {
   defer func() {
      if err := recover(); err != nil {
         fmt.Println(err)
      } else {
         fmt.Println("fatal")
      }
   }()

   defer func() {
      panic("defer panic")
   }()
   panic("panic")
}

func Test_demo21(t *testing.T) {
   defer func() {
      if err := recover(); err != nil {
         fmt.Println("++++")
         f := err.(func() string)
         fmt.Println(err, f(), reflect.TypeOf(err).Kind().String())
      } else {
         fmt.Println("fatal")
      }
   }()

   defer func() {
      panic(func() string {
         return "defer panic"
      })
   }()
   panic("panic")
}
```

#### 答案

<details>
=== RUN   Test_demo20
panic
--- PASS: Test_demo20 (0.00s)
</details>


<details>
=== RUN   Test_demo21
++++
0x1103140 defer panic func
--- PASS: Test_demo21 (0.00s)
</details>


panic仅有最后一个可以被recover捕获