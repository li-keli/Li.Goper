---
title: "Go_No4"
date: 2020-03-01T09:29:11+08:00
draft: false
tags: [Go, 刷题]
featured_image: "https://oss.likeli.top/uPic/20200401094728"
categories: 刷题
---

## go 试卷 NO.4

### 执行下面代码发生什么？

```go
type Param map[string]interface{}

type Show struct {
   *Param
}

func main() {
   s := new(Show)
   s.Param["RMB"] = 10000
}
```

#### 答案

![20200305140110](https://oss.likeli.top/uPic/20200401094111)

map在使用之前需要进行初始化



### 执行下面代码会发生什么？

```go
type student1 struct {
   Name string
}

func JJ(v interface{}) {
   switch msg := v.(type) {
   case *student1, student1:
      msg.Name = "qq"
      fmt.Print(msg)
   }
}
```

#### 答案

msg不属于student1类型，所以不存在Name字段



### 执行下面的代码会发生什么？

```go
type People struct {
   name string `json:"name"`
}

func Test_demo23(t *testing.T) {
   js := `{
        "name":"11"
    }`
   var p People
   err := json.Unmarshal([]byte(js), &p)
   if err != nil {
      fmt.Println("err: ", err)
      return
   }
   fmt.Println("people: ", p)
}
```

#### 答案

结构体中`name`字母小写为私有，所以无法反序列化赋值



### 下面的代码有什么问题？

```go
func Stop(stop <-chan bool) {
    close(stop)
}
```

#### 答案

关于`close`

* 关闭一个nil的Channel或者已经关闭的Channel会抛出运行时异常
* 带有方向的Channel不能被Close



### 下面的代码输出什么？

```go
func Test_demo24(t *testing.T) {
	five := []string{"Annie", "Betty", "Charley", "Doug", "Edward"}

	for _, v := range five {
		five = five[:2]
		fmt.Printf("v[%s]\n", v)
	}

	fmt.Println(five)
}
```

#### 答案

<details>
=== RUN   Test_demo24
v[Annie]
v[Betty]
v[Charley]
v[Doug]
v[Edward]
[Annie Betty]
--- PASS: Test_demo24 (0.00s)
</details>


range的副本机制。

循环内的切片值会缩减为2，但循环将在切片值的自身副本上进行操作。也就是循环使用原始长度进行迭代。

但是外部five切片已经被修改



### 输出MultipleParam= [ssss [1 2 3 4]]如何做到输出为[ssss 1 2 3 4]

```go
func MultipleParam(p ...interface{}) {
   fmt.Println("MultipleParam=", p)
}

func Test_demo25(t *testing.T) {
   MultipleParam("ssss", 1, 2, 3, 4)
   iis := []int{1, 2, 3, 4}
   MultipleParam("ssss", iis)
}
```

#### 答案

**函数可变参数**

方法一：使用[]interface

```go
func Test_demo25(t *testing.T) {
   iis := []int{1, 2, 3, 4}
   newParam := make([]interface{}, 0)
   newParam = append(newParam, "ssss")
   for _, v := range iis {
      newParam = append(newParam, v)
   }
   MultipleParam(newParam...)
}
```

方法二：反射

```go
func Test_demo26(t *testing.T) {
   iis := []int{1, 2, 3, 4}
   f := MultipleParam
   value := reflect.ValueOf(f)
   pps := make([]reflect.Value, 0, len(iis)+1)
   pps = append(pps, reflect.ValueOf("ssss"))
   for _, ii := range iis {
      pps = append(pps, reflect.ValueOf(ii))
   }
   value.Call(pps)
}
```

[相关文档](http://c.biancheng.net/view/113.html)



#### 补充

go前三个点与后三个点的意思：

* 做为形参的前三个点意思是可以穿0到n个参数
* 变量后三个点意思是将一个切片或数组变成一个个的元素，俗称将数组打散



### 实现一个函数可以根据指定的size切割切片为多少个小切片

#### 答案

```go
https://studygolang.com/articles/14871
```



### 实现两个go轮流输出：A1B2C3.....Z26

#### 答案

```go

```



