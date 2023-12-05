---
title: Go语言基础语法
date: 2021-12-23 10:09:15
tags:
    - Go
categories:    
    - Go
description: 我们做了大量的 C++ 开发，厌烦了等待编译完成，尽管这是玩笑，但在很大程度上来说也是事实
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/golang.png
---
## Go的起始
    我们做了大量的 C++ 开发，厌烦了等待编译完成，尽管这是玩笑，但在很大程度上来说也是事实。
### Go是编译型语言
Go使用编译器来编译代码。Go自带编译器，所以，我们只需要写好源码就行了。一般，都是如下步骤：
1. 使用文本编辑器创建go程序
2. 保存文件
3. 编辑程序
4. 运行编译成对应系统的可执行文件

和c++较量编译快，和Java较量执行高效，和Python较量开发快速。GO在三个语言中做了平衡，获得了自己的优势：快速编译，高效执行，快速开发。

### 特性
1. 语法简单
2. 并发模型
3. 内存分配
4. 垃圾回收
5. 静态链接
6. 标准库
7. 工具链

### 为并发而生
Go语言从底层执行并发，无需第三方库，开发人员可以很轻松地在编写程序时决定怎么使用CPU资源。Go并发基于`goroutine`。
Go在运行时会调用goroutine，并将goroutine合理地分配到每个CPU上，最大限度地使用CPU性能。多个goroutine之间，使用`channel`通信。

### 编译与运行
Go语言是编译型的静态语言，所以在运行之前，需要将其编译为二进制的可执行文件。

`go build`命令可以将Go语言代码编译为二进制的可执行文件，但是需要我们手动执行该文件
`go run`命令则会自动编译后执行该文件，是集合编译+执行的命令，但是，它不会生成可执行文件，只会生成一个临时文件，可用于调试

## Go的语法
    Go 语言在很多特性上和C语言非常相近
### Go的变量
Go语言的**基本类型**有:
1. bool
2. string
3. int、int8、int16、int32、int64
4. uint、uint8、uint16、uint32、uint64、uintptr
5. byte // uint8 的别名
6. rune // int32 的别名 代表一个 Unicode 码
7. float32、float64
8. complex64、complex128 // 复数

Go语言**变量声明**：
1. 标准格式：`var name type`，类型在后，变量名在前，使用var来声明
2. 简单格式：`name := 表达式`

**初始化**：go中不存在未初始化的变量
1. 整型和浮点型变量的默认值为 0 和 0.0
2. 字符串变量的默认值为空字符串
3. 布尔型变量默认为 bool
4. 切片、函数、指针变量的默认为 nil

{% codeblock lang:go %}
// 可以不需要明确类型，编译器会自动推断类型
标准格式：var 变量名 类型 = 表达式; var a int = 100
简单格式：变量名 := 表达式; a := 100
{% endcodeblock %}

**匿名变量**：
我们使用`_`来表示匿名变量，称为空白标识符。匿名变量的值都将会抛弃，后续代码中将不再使用。这样可以极大的增强代码的灵活性。
{% codeblock lang:go %}
func GetData() (int, int) {
    return 100, 200
}
func main(){
    a, _ := GetData()
    _, b := GetData()
    fmt.Println(a, b)
}
{% endcodeblock %}

**字符类型**：两种(byte & rune)
1. byte: 代表了一个ASCII码的一个字符，相当于uint8
2. rune: 代表一个UTF8字符，当需要处理中文等其他复合字符时，需要用到rune类型。rune类型相当于int32类型
**字符串类型**：一个不可改变的字节序列。可以通过`len`函数来得到字符串的长度。非ASCII字符的UTF8编码，字节长度可能是2个及其以上个字节长度。
   
标准库中有四个包对字符串处理尤为重要：
1. strings: 字符串的截取，查询，比较等
2. bytes: 与上面功能类型，对byte类型数据处理 
3. strconv: 对其他基础类型的转换
4. unicode: 给字符分类

{% codeblock lang:go %}
    // string与byte相互转换
    s := "abc"
	b := []byte(s)
	s1 := string(b)
{% endcodeblock %}
**类型转换**：只能在类型显式声明的情况下进行转换，大转小，精度缺失。只有相同底层类型的变量之间才能相互转换。常用转换方法如下：
`func Itoa(i int) string`
`func Atoi(s string) (i int, err error)`
**Parse 系列函数**：Parse 系列函数用于将字符串转换为指定类型的值
{% codeblock lang:go %}

    func ParseBool(str string) (value bool, err error)
    func ParseInt(s string, base int, bitSize int) (i int64, err error)
    func ParseUint(s string, base int, bitSize int) (n uint64, err error)
    func ParseFloat(s string, bitSize int) (f float64, err error)
{% endcodeblock %}
**Format 系列函数**：Format 系列函数实现了将给定类型数据格式化为字符串类型的功能
{% codeblock lang:go %}

    func FormatBool(b bool) string
    func FormatInt(i int64, base int) string
    func FormatUint(i uint64, base int) string
    func FormatFloat(f float64, fmt byte, prec, bitSize int) string
{% endcodeblock %}
**Append 系列函数**：Append 系列函数用于将指定类型转换成字符串后追加到一个切片中

**常量**：使用关键字const定义，只能是布尔型，数字型和字符串
**iota常量生成器**：使用批量声明常量的规则，用于生成一组以相似规则初始化的常量。第一个声明const的常量处所在的行，iota会被置为0，然后在每一个有常量声明的行加一。
这种方式，也可以被称为枚举。
{% codeblock lang:go %}

    type Weekday int
    
    const (
        Sunday Weekday = iota // 0，之后会依次递增
        Monday
        Tuesday
        Wednesday
        Thursday
        Friday
        Saturday
    )
{% endcodeblock %}

简单的类型判断：`s.(type)` s代表要判断的参数，这必须用在switch结构语句上才行

### 指针
    指针是一种直接存储了变量的内存地址的数据类型
#### 类型指针
    类型指针：允许对这个指针类型的数据进行修改，传递数据可以使用指针，而无需拷贝指针，指针类型不能是偏移和运算

#### 切片
    切片：由指向起始元素的原始指针、元素数量和容量组成

指针是一个变量，其值为另一个变量的内存地址。每个变量都有自己的内存地址。它的零值为nil。
{% codeblock lang:go %}

    name := &v // v的类型为T
    t := *name // 获取指针变量name指向的值
{% endcodeblock %}

如上：使用了`&`关键字获取变量`v`的内存地址，并将其赋予变量`name`，`name`的类型为`*T`，称为T的指针类型，`*`代表指针。

`&`和`*`是一对互补操作符。`&`获取内存地址，`*`获取内存地址指向的值。

变量、指针地址、指针变量、取地址、取值的相互关系和特性如下：
- 对变量进行取地址操作使用`&`操作符，可以获得这个变量的指针变量。
- 指针变量的值是指针地址。
- 对指针变量进行取值操作使用`*`操作符，可以获得指针变量指向的原变量的值。

**使用指针修改值**
{% codeblock lang:go %}

    func main() {
    
        x, y := 1, 2
        swap1(&x, &y)
        println(x, y)
    
        println("--------------------------")
    
        j, k := 1, 2
        swap2(&j, &k)
        println(j, k)
    }
    
    // 交换指针变量指向的值
    func swap2(a, b *int)  {
        println(*a)
        println(*b)

        t := *a
        *a = *b
        *b = t
    
        println(*a)
        println(*b)
    }
    // 交换指针变量的值
    func swap1(a, b *int)  {
        println(a)
        println(b)
    
        a,b = b,a
    
        println(a)
        println(b)
    }
{% endcodeblock %}
结果：
```text
0xc00003df58
0xc00003df50
0xc00003df50
0xc00003df58
1 2
--------------------------
1
2
2
1
2 1
```
结论：
1. `swap1()`方法交换指针变量的值，a和b并没有变换，只有其内存地址进行了交换
2. `swap2()`交换指针变量指向的值，a和b进行了交换，而且其内存地址也进行的交换

`*`操作符**归纳**：`*`操作的是指针地址指向的变量。当操作的值在`=`右边时，获取指针变量指向的值，当操作的值在`=`左边时，就是将值设置给指针指向的变量

#### 变量的生命周期
变量的生命周期和变量的作用域相关，变量主要有三个作用域：
1. 全局变量：随着程序的生命周期开始和结束
2. 局部变量：开始于使用这个变量，结束不用这个变量，会被垃圾回收
3. 形参和函数返回值：特殊的两种变量，属于局部变量，随着方法调用被创建，方法调用结束被销毁

堆栈：
- 堆：用于存放进程中被动态分配的内存段，大小不固定，可以动态扩张或缩减。
- 栈：用来存放暂时创建的局部变量

垃圾回收：
go语言垃圾回收的判断，会首先根据变量的作用域来判断，全局变量跟随程序，那么只会在程序执行完成进行回收，而且变量存在堆中。
局部变量的话，它可以通过=某个路径获取到这个局部变量，如果不存在这个路径，那么该变量也就没人使用，可以进行垃圾回收了。
总而言之，变量作为源头，无论是全局或者局部变量，都可以通过指针获取其他方式的引用，通过某个路径找到它。如果路径不存在，那么就判断为不可访问，可以垃圾回收。

## Go的数组及切片
    Go的语言容器，可以存储复数的相同类型元素

### Go的数组
    数组拥有固定的长度，并由特定的元素类型组成
声明一维语法：`var 数组变量名[长度]Type`
声明多维语法：`var 数组变量名[长度1][长度2]...[长度n]Type`
{% codeblock lang:go %}

    // 比较数组：长度类型皆相同才相等
    a := [2]int{1, 2}
    b := [...]int{1, 2}
    c := [2]int{1, 3}
    fmt.Println(a == b, a == c, b == c) // true false false
    d := [3]int{1, 2}
    fmt.Println(a == d) // 编译错误：无法比较 [2]int == [3]int
    // 多维数组
    
{% endcodeblock %}

### 切片(Slice)
    与数组对应，它的长度不是固定的，可以动态的增加或者减少，如同Java的集合一样。它是对数组的一个连续片段的引用，属于引用类型
**内部结构**：地址，大小和容量。切片并不是一个单纯的引用类型，它实际如同下面的聚合类型一样
{% codeblock lang:go %}
type InitSlice struct {
    ptr *int
    len, cap int
}
{% endcodeblock %}
**初始化切片**：

1. 从数组或切片生成新的切片。`slice [开始:结束]`

{% codeblock lang:go %}
    
    a := [3]int{1,2,3} // 数组
    b := a[1,2] // [2] 切片，从1位置开始，2位置结束，不包含2
    c := a[1:] // [2,3], 结束位置缺省，获取开始位置，到之后所有元素
    d := a[:2] // [1,2], 开始位置缺省，获取结束位置之前所有元素
    aa := []int{4,5,6} // 切片
    e := aa[:] // [4,5,6],生成与原切片一样的新切片  
    f := aa[0:0] // [],空切片

{% endcodeblock %}

2. 声明新切片：`var name []Type`

{% codeblock lang:go %}
    
    // 声明字符串切片
    var strList []string
    // 声明整型切片
    var numList []int
    // 声明一个空切片,由于已经初始化，内存空间已经开辟
    var numListEmpty = []int{}
    // 输出3个切片
    fmt.Println(strList, numList, numListEmpty) // [] [] []
    // 输出3个切片大小
    fmt.Println(len(strList), len(numList), len(numListEmpty)) // 0 0 0
    // 切片判定空的结果
    fmt.Println(strList == nil) // true
    fmt.Println(numList == nil) // true
    fmt.Println(numListEmpty == nil) // false
{% endcodeblock %}

3. 使用`make()`函数创建：`make([]Type, size, cap)`

{% codeblock lang:go %}

    a := make([]int, 2)
    b := make([]int, 2, 10)

    fmt.Println(a, b) // [0,0] [0,0]
    fmt.Println(len(a), len(b)) // 2 2, 虽然容量为10，但只使用了2个位置，所以长度为2
{% endcodeblock %}
    注：`make()`生成的切片一定有内存分配操作，但给定开始与结束位置的切片只是将新的切片结构指向已经分配好的内存区域，所以，设定开始与结束位置，并不会发送内存分配

#### 操作切片
`append()`: 为切片添加元素。当空间不足时，会动态扩容，扩容规律为容量的2的倍数扩容。1，2，4，8...32...2n。我们不能确定原先的切片是否会影响到新的切片，所以需要将返回值赋值给输入的切片变量。
{% codeblock lang:go %}
    
    var a []int
    a = append(a, 2) // 追加1个元素
    a = append(a, 2, 3) // 追加多个元素, 手写解包方式
    a = append(a, []int{1,2,3}...) // 追加一个切片, 切片需要解包
    a = append([]int{1}, a...) // 在a切片前面加一个新元素，会重新构建切片，性能较差
{% endcodeblock %}
    
`copy(destSlice, srcSlice []T) int`: 切片复制。复制的切片类型必须一致。destSlice：源切片；srcSlice：目标切片
**删除切片元素**：利用切片的特性删除，但是很耗性能
{% codeblock lang:go %}
    
    a = int[]{1,2,3,4} 
    a = a[2:] // 删除开头2个元素
    a = append(a[:1], b[2:]...) // 删除中间2下标元素
    a = append(:len(a) - 2) // 删除尾部2个元素
{% endcodeblock %}

`range关键字`：循环迭代切片
{% codeblock lang:go %}

    a = int[]{1,23,4,56}
    for index,value := range a {
        fmt.Printf("Index: %d Value: %d\n", index, value) // index为对应索引，value为对应元素值的副本，非元素本身
    }
{% endcodeblock %}

### map
字典结构，hash结构，是一种key-value的结构，长度也是动态的，未初始化时未nil，`len()`可以活动长度。遍历map也可以使用`range`关键字
`var mapname map[keytype]valuetype`

{% codeblock lang:go %}
var e map[int64]string // 第一种声明方式
e = map[int64]string{1: "one", 2: "tow"}
fmt.Println(e) // map[1:one 2:tow]

f := make(map[string]int) // 第二种声明方式，不能使用new()来声明
f["a"] = 1
f["b"] = 2
fmt.Println(f) // map[a:1 b:2]
{% endcodeblock %}
    
#### delete(map, 键) 
删除map键值对。如果需要清空，使用make()函数重新创建一个map

#### sync.Map：并发环境下map
并发环境下，go只能保证并发读取，并不能保证并发写入，如需要并发写入，需要加锁，但是效率并不高，自1.9版本之后，go引入了sync.Map类，来保证并发写入。
**特性**：
1. 无须初始化，直接声明
2. 不能使用map的相关方法，需要使用自己的函数进行数据的读写
3. 使用Range配置一个回调函数进行变量操作，回调函数中，返回true继续遍历，返回false停止遍历

{% codeblock lang:go %}
var syncMap = sync.Map{}
syncMap.Store("str1", "str2")
syncMap.Store("str2", "str3")
syncMap.Store("str3", "str4")

fmt.Println(syncMap.Load("str3")) // str4 true；可以查到，返回对应的值
fmt.Println(syncMap.Load("str4")) // nil false；不存在，返回nil

syncMap.Delete("str4")

syncMap.Range(func(key, value interface{}) bool { // 遍历
    fmt.Println("iterate:", key, value)
    return true
})
{% endcodeblock %}

注：并没有对应计算长度的方法，需要遍历获取长度，性能也不如非并发map

### list: 列表
列表是一种非连续的存储容器，由多个节点组成，节点通过一些变量记录彼此之间的关系，列表有多种实现方法，如单链表、双链表等。
**初始化**：
1. 通过container/list包的`New()`函数初始化list
   `变量名 := list.New()`
2. 通过`var`关键字声明初始化list
   `var 变量名 list.List`
**新增&删除**

{% codeblock lang:go %}
l := list.New()
// 在前面添加
l.PushFront("sss")
// 在后面添加
l.PushBack(111)
// 在某个元素之后插入
l.InsertAfter("after", els)
el := l.PushBack("back")
// 删除对应元素
l.Remove(el)
// 遍历元素
for i := l.Front(); i != nil; i = i.Next() {
    fmt.Println("list value", i.Value)
}
// 获取长度
fmt.Println("len", l.Len())
{% endcodeblock %}

#### nil: 空值/零值
在Go语言中，布尔类型的零值（初始值）为 false，数值类型的零值为 0，字符串类型的零值为空字符串""，而指针、切片、映射、通道、函数和接口的零值则是 nil。
nil与其他语言的null有类似的地方，但也有诸多不同的地方，如下便是nil的独特性：
1. nil标识符不能与自己比较
2. nil不是关键字或保留字
3. nil没有默认类型
4. 不同类型的nil指针一致
5. 不同类型的nil不能比较
6. 相同类型的nil也可能无法比较
7. nil 是 map、slice、pointer、channel、func、interface 的零值
8. 不同类型的nil所占的内存大小可能不一致

下面，我们就一一验证这些特性 

{% codeblock lang:go %}
// 1.
fmt.Println(nil == nil) // invalid operation: nil == nil (operator == not defined on nil)
// 2. 
nil := list.New()
nil.PushBack("ss")
fmt.Println(nil.Len()) // 1
// 3.
fmt.Printf("%T", nil) // <nil>
print(nil) // use of untyped nil
// 4.
var slice []int
fmt.Printf("%p\n", slice) // 0x0

var maps map[int64]string
fmt.Printf("%p", maps) // 0x0
// 5.
fmt.Println(slice == maps) // Invalid operation: slice == maps (mismatched types []int and map[int64]string)
// 6.
var copySlice []int
fmt.Println(slice == copySlice) // Invalid operation: slice == copyslice (the operator == is not defined on []int)
// 7.
fmt.Printf("%#v\n", slice) // []int(nil)
fmt.Printf("%#v\n", maps) // map[int64]string(nil)
// 8. 具体的大小取决于编译器和架构，下面打印的结果是在64位架构和标准编译器下完成的，对应32位的架构的，打印的大小将减半
fmt.Println(unsafe.Sizeof(slice)) // 24
fmt.Println(unsafe.Sizeof(maps)) // 8
{% endcodeblock %}
