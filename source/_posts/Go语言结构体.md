---
title: Go语言结构体
date: 2021-12-29 17:20:30
tags:
    - Go
categories:    
    - Go
description: 在go语言中，并没有”类“的概念，只有结构体和接口，官方认为通过结构体和接口的组合，能够更加灵活和更具扩展性
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/golang.png
---
# Go的结构体与函数
在go语言中，并没有”类“的概念，只有结构体和接口，官方认为通过结构体和接口的组合，能够更加灵活和更具扩展性

## 结构体
结构体成员是由一系列的成员变量构成，这些成员变量也被称为“字段”。字段有以下特性：
1. 字段拥有自己的类型和值。
2. 字段名必须唯一。
3. 字段的类型也可以是结构体，甚至是字段所在结构体的类型。
4. 结构体不能包含自己类型的成员，但是可以包含自己类型的指针类型(指针，链表等数据结构)
5. 结构体的全部成员都是可以比较的，结构体也是可以比较的，可以使用`==`或`!=`比较

```text
type 类型名 struct {
    字段1 字段1类型
    字段2 字段2类型
    …
}
// 包含自己类型指针
type tree struct {
    left, right *tree
    value int
}
```

### 实例化
可以通过`var`来实例化结构体: `var instance T`
也可以通过`new`来实例化: `instance := new(T)`
也可以通过获取结构体的指针地址来进行实例化: `instance := &T{}`

{% codeblock lang:go %}
var user1 User
user1.Age = 10
user1.Name = "var"
fmt.Println("user1", user1)

user2 := new(User)
user2.Name = "new"
user2.Age = 12
fmt.Println("user2", user2)

user3 := &User{"pointer", 14}
fmt.Println("user3", user3)    
{% endcodeblock %}

#### 初始化匿名结构体
匿名结构体没有类型名称，无需通过type关键字定义就可以直接使用
```text
匿名结构体格式：
ins := struct {
    字段1 字段类型1
    字段2 字段类型2
    …
}
匿名结构体初始化：
ins := struct {
    // 匿名结构体字段定义
    字段1 字段类型1
    字段2 字段类型2
    …
}{
    // 字段值初始化
    初始化字段1: 字段1的值,
    初始化字段2: 字段2的值,
    …
}
```
{% codeblock lang:go %}
user4 := struct{
    Name string
    Age int64
}{
    Name: "user4",
    Age: 18,
}
fmt.Println("user4", user4)
{% endcodeblock %}

#### 结构体内嵌
如下结构体包含匿名字段和普通字段，还有内嵌的结构体，相同类型的匿名字段只能包含一个。
特性：
1. 内嵌的结构体可以直接访问其成员变量：提高访问效率，如果内嵌结构体有相同名称字段，需要全写，不能简写
2. 内嵌结构体的字段名是它的类型名：也可以一层层的去访问内嵌结构体，结构体就是它的类型名

{% codeblock lang:go %}
   
    type User struct {
        Name string
        Age int64
        *Child
        int
        string
        Parent
    }

    type Parent struct {
        Name string
        Age int64
    }

    type Child struct {
        Name string
        Age int64
    }
    
    // 实例化结构体，匿名结构体可以直接使用类型名作为字段名
    user5 := new(User)
    user5.Name = "user5"
    user5.Age = 22
    user5.string = "string"
    user5.int = 1
    user5.Child = &Child{Name: "child"}
    user5.Parent = Parent{Name: "parent"}
    
    fmt.Println("user5", user5)
    fmt.Println("user5.Parent", user5.Parent)
    // 省略中间结构体，直接访问内部变量
    fmt.Println("user5.Parent", user5.Parent)
    // 可以访问内嵌的结构体，内嵌结构体甚至可以来自其他包。结构体就是它的字段名
    fmt.Println("user5.child.name", user5.Child.Name)
{% endcodeblock %}

## 函数
go里面包含两种类型的函数：
1. 普通函数
```text
func 函数名(形式参数列表)(返回值列表){
    函数体
}
```
2. 匿名函数
```text
func(参数列表)(返回参数列表){
   函数体
}
```
函数也是一种类型，可以进行赋值操作，作为一个变量进行使用。
匿名函数可以作为回调函数使用，也可以作为普通函数使用。也就是说，可以作为类型赋值给变量，也可以直接调用匿名函数
{% codeblock lang:go %}
type Parent struct {
    Name string
    Age int64
    Add func(Age int64) int64 // 作为函数类型，内嵌结构体
}

func func01() (a, b int) {
    a = 1
    b = 2
    return
}
func func02(a, b int) int {
    return a + b
}
// 函数作为类型，进行赋值给变量
f1 := func01
f1()

// 匿名函数直接调用
func(data int64) {
    fmt.Println("hello", data)
}(100) // 作为参数传递给匿名函数

// 匿名函数赋值给变量
f6 := func(age int) {
    fmt.Println("age", age)
}
f6(100)
{% endcodeblock %}

### defer
Go语言中，`defer`关键字是用于延迟处理。如果有多个defer关键字，最后被修饰的最先执行。多用于释放资源等操作

{% codeblock lang:go %}
defer fmt.Println("defer1")
defer fmt.Println("defer2")
defer fmt.Println("defer3")
// 输出顺序：defer3 defer2 defer1
{% endcodeblock %}

### panic
Go语言可以在程序中手动触发宕机，让程序崩溃，这样开发者可以及时地发现错误，同时减少可能的损失。可以和defer结合使用，在调用`panic()`函数的时候，之前`defer`关键字修饰的方法可以依次执行在宕机之前

### recover
Recover 是一个Go语言的内建函数，可以让进入宕机流程中的goroutine恢复过来，recover仅在延迟函数defer中有效，在正常的执行过程中，调用recover会返回nil并且没有其他任何效果，如果当前的goroutine陷入恐慌，调用 recover 可以捕获到 panic 的输入值，并且恢复正常的执行。
#### panic和recover的关系
panic 和 recover 的组合有如下特性：
- 有 panic 没 recover，程序宕机。
- 有 panic 也有 recover，程序不会宕机，执行完对应的 defer 后，从宕机点退出当前函数后继续执行。
不推荐使用

### 函数执行时间
```text
func Since(t Time) Duration
```
{% codeblock lang:go %}
now := time.Now()
sum := 0
for i := 0; i < 100000; i++ {
    sum++
}
elapsed := time.Since(now)
fmt.Println("耗时：", elapsed)
{% endcodeblock %}

### Test测试
我们需要对我们写的代码进行测试，来保证程序准确并安全的运行。go提供测试的相关规则：
1. 测试文件已`_test.go`结尾
2. 测试文件内可以写多个测试方法
3. 测试方法需要与被测试的方法一一对应，测试函数的名称要以Test或Benchmark开头，后续的方法名第一次字母必须为大写。
4. 测试用例不会参与正常源码编译
5. 单元测试则以(t *testing.T)作为参数，性能测试以(t *testing.B)做为参数
6. 需要使用 import 导入 testing 包

```text
单元测试
func TestXxx( t *testing.T ){
    //......
}

性能测试
func TestXxx( t *testing.B ){
    //......
}

覆盖率测试
func TestXxx( t *testing.T ){
    //......
}
func TestXxx( t *testing.B ){
    //......
}
```
## JSON
    JavaScript对象表示法（JSON）是一种用于发送和接收结构化信息的标准协议。

{% codeblock lang:go %}
type Person struct {
    Height int64
    Age int
    Name string
}

var p = Person{
    1,1,"xin"
}
// 转换为json
data, err := json.Marshal(p) 
if err != nil {
    log.Fatalf("JSON marshaling failed: %s", err)
}

// 解析字符串为结构体
var p2 Person
if err := json.Unmarshal(data, &p2); err != nil {
    log.Fatalf("JSON unmarshaling failed: %s", err)
}
{% endcodeblock %}
## 文本和HTML模板

