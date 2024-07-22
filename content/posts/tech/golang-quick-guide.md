---
title: "GO 语言快速入门"
date: 2020-08-16T21:47:07+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "golang"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "提供给有编程基础、希望快速了解 Golang 的朋友清晰简洁的基础知识点，帮助他们快速入门这门语言。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---

## 00 Go 环境配置

首先，下载 Golang 安装包，地址：https://go.dev/dl/。

接着，将 Golang 安装包解压到指定路径。

然后，把 Go 安装路径配置到环境变量。有如下两个环境变量：

- GOROOT：可选，主要是将 `$GOROOT/bin` 配置到环境变量中以便直接使用 go 命令及其工具等
- GOPATH：Go 1.11 之后引入 Go Modules，不再完全依赖 `GOPATH` 了
    - Go 1.11 之前，`go get` 下载的包会被存放在 `$GOPATH/src` 目录下，与项目无关
    - Go 1.11 之后（使用了 Go Modules），依赖包会被存放在项目的 `go.mod` 指定的缓存目录中，默认情况下是 `$GOPATH/pkg/mod`

## 01 Go 基础

### 1.1 变量

1. 类型：放在变量名后面
2. 变量声明：`var NAME TYPE`
3. 简单变量声明：`NAME := VALUE`（只允许在函数内部使用，且不能用于声明静态变量）
4. 数据类型

```go
bool

string  // 不出初始化，则默认空字符串：""

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128 // 复数
    var x complex128 = complex(1, 2) // 1+2i
    var y complex128 = complex(3, 4) // 3+4i
    fmt.Println(x*y)                 // "(-5+10i)"
    fmt.Println(real(x*y))           // "-5"
    fmt.Println(imag(x*y))           // "10"
```

### 1.2 循环

Go 只有 `for` 循环，且格式固定：

```go
// 没有小括号，大括号必须有
for i := 0; i < 10; i++ {
    // do ...
}

// Go 的 while 是 for 的特例
for s < 1000 {
    // do ...
}

// 忽略其他条件，死循环，即 while(true)
for {
    // do ...
}
```
    
### 1.3 条件判断
    
Go 的条件判断有 `if` 和 `switch`。`if` 的区别是：

1. 可以省略小括号
2. 可以有初始语句，类似 `for` 循环的初始化变量语句

```go
// 可以省略小括号，大括号必须有
if x < 0 {
    // do
}

// 还可以有多语句
if v := f(x, n); v < n {
    // do
}
```

`Switch` 的区别比较大，它只执行单个 `case`，然后自动 `break`

- 默认 `break`，如果需要继续执行，则须使用 `fallthrough`
- 条件可以为空：相当于 `switch true`，可以代替很长的 `if else`
    
```go
switch os := runtime.GOOS; os {
    case "darwin":
        // do
    case "linux":
        // do
        fallthrough
    default:
        // do
}

// if else
switch {
    case x < 0:
        // if x < 0
    case x < 2:
        // if x < 2
    default:
        // else
}
```
    
### 1.4 延迟调用（defer）
    
Go 比 C/Java 多了延迟调用函数 `defer`。延迟调用的函数的**参数**会立即计算，但函数在当前函数 `return` 时（结束）才执行。

- `defer` 函数入栈，故多个延迟函数将**倒序**执行。
- 另一个作用：配合 `recover()` 捕获异常。

```go
// x = 1, y = 1
func test(x, y int) int {
	defer my_defer(&x, y+1)         // *param1 = 1, param2 = 2
	fmt.Println("test: x = ", x)    // x = 1，x 的值如果修改，defer 的 x 将同步修改！
	return x
}
```
    
### 1.5 指针与结构体
    
Go 有指针，但没有指针运算（C 中的 `int *p = &arr; p++; ...`）

```go
var p *int
i := 222
p  = &i
*p = 22
```

Go 和 C 一样都有结构体，但它可以指定变量初始化，初始化方式稍微有些差异：

```go
type Vertex struct {
    x, y int
}

var (
    v1 = Vertex{1, 2}
    v2 = Vertex{y: 3}
    p  = &Vertex{1,1}  // p.x == 1, p.y == 1
)
```
    
### 1.6 切片（Slice）
    
Go 的切片 `Slices` 和 JavaScript 的 Array 有相似之处

- 切片不存储数据，仅指向数组的位置，数组改变切片也会改变。
- 切片有长度和容量：`len(s)` and `cap(s)`（注意左边的取值会影响容量）

```go
// 左闭右开区间
primes := [6]int{2, 3, 4, 5, 6}
// s = {3, 4, 5}
var s []int = primes[1:4]

// 可省略左右区间值：左-0，右-length
var s = primes[:2] // 2,3
var s = primes[2:] // 4,5,6
var s = primes[:]  // 同原数组

fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)

// 可以看作创建了一个空的数组，s 是指向这个数组的切片，则 s == nil
var s []int

// 遍历
for INDEX, ELEMENT := range SLICE {
    // DO...
}
```

动态创建切片，使用 `make` 预分配空间。在实际使用的过程中，如果容量不够，切片容量会自动扩展

```go
var s = make([]int, 5)    // len=5 cap=5 [0,0,0,0,0]
var s = make([]int, 0, 5) // len=0 cap=5 []
```

添加到切片，使用 `append` 添加/合并，切记不要合并不同的两个切片

```go
// func append(s []T, vs ...T) []T
var s = append(s, 1, 2)      // [[s], 1, 2]

// Do not do this!!!
// 1. 容量不够，会创建新切片
someSlice = append(otherSlice, element)
// 2. 容量够，会直接加入，多次创建新的会覆盖旧的
a := make([]int, 3, 8)
b := append(a, 5) // b = [0, 0, 0, 5]
c := append(a, 6) // b = [0, 0, 0, 6]
                  // c = [0, 0, 0, 5]
```
    
### 1.7 可变参数与展开操作符

Go 没有明确的 Spread Operator 操作符，而 `...` 可以将切片展开为可变参数列表

```go
package main

import "fmt"

func variadicParam(nums ...int) int {
	// nums is a slice
	total := 0
	for i := 0; i < len(nums); i++ {
		total += nums[i]
	}

	return total
}

func main() {
  // variadic parameter
  fmt.Printf("Sum of any 1: %v\n", variadicParam(1, 2, 3)) // 6

  // spread operator
  data := []int{1, 3, 5, 7}
  fmt.Printf("Sum of any 2: %v\n", variadicParam(data...)) // 16
}
```

### 1.8 范围（range）

在 Go 语言中，range 关键字用于迭代数组、切片、通道（channel）、字符串或映射（map）等数据结构中的元素

```go
var pow = []int{1, 2, 4, 8, 16}
// 返回：(index, value)
for i, v := range pow {
    // do v = 1, 2, ...
}
```

并且，可以任意忽略索引或者值

```go
for i, _ := range pow {
    // do
}
for _, v := range pow {
    // do
}
// 只要索引
for i := range pow {
    // do
}
```

### 1.9 字典（Map）

Go 的 Map 与 java 的 HashMap 类似。

```go
// 1. 仅声明
m := make(map[string]int)
// 2. 声明并初始化
m := map[string]string {
    "key": "value",
}
m["key"] = "haha"

// 3. Map 多层嵌套
m := make(map[string]map[string]int)
// m: {
//     "first": { "v": 1, ... },
//     "second": { "v": 2, ... },
// }
```

### 1.10 枚举

Go 没有枚举类型，可以用 `const` 定义

- `type`：和 `typedef` 一样

```go
type Gender uint8
const (
    MALE   Gender = 1
    FEMAIL Gender = 0
)
```

### 1.11 错误处理（error）

Go 内置 `error`，类型即为 `error`：

```go
type error interface {
    Error() string
}
```

自定义错误只需 `error.New("MSG")` 直接 New 一个错误即可，`error` 只是一个值，把它看错 C 语言里的 `return -1/0/1` 就好。

可以使用 `defer` 捕获异常，使用 `recover` 恢复，配合使用达到 `try...catch` 的效果：

- `defer`：延迟函数
- `recover`：程序恢复正常

```go
func get(i int) (ret int) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println(r)
			ret = -1  // 程序恢复正常，并且将返回值设置为 -1（不处理默认为 0）
		}
	}()
	y := i / (i - 2)
	return 22 + y;
}

fmt.Println(get(2))
// runtime error: integer divide by zero
// -1
```

### 1.12 函数（func）

函数使用 `func` 声明：

- 格式：`func functionName(parameter TYPE) TYPE`
- 特点：可以返回多个参数，无需创建对象/数组再组合返回

```go
// 1. 普通函数
func f1(a int, b int) int {}
func f1'(a, b int) int {}

// 2. 多返回值
func f2() (string, int) {
   return "OK", 0
}
r, ok := f2()
r, _  := f2() // 忽略对应值

// 3. 多返回值 - 命名返回变量，自动返回对应值
func f3() (x, y int) {
   var x int
   var y int
   return // 自动返回 x,y
}

func f3'() (x, y int) {
   return 1, 2  // 返回值覆盖，x, y 不会被返回
}
```

### 1.13 结构体和方法（struct）

#### 结构体和方法的定义

Go 的结构体使用 `type NAME struct` 定义

```go
// 定义结构体
type Student struct {
    name string
    age int
}

// 定义方法：下面都可以，如果需要调用该类的字段，则需要声明变量
//      * 定义结构体类型：只读
//      * 定义结构体指针：可写
// func (Student) hello(person string) string
// func (*Student) hello(person string) string
func (stu *Student) hello(person string) string {
    return "Hello " + person + ". I am" + stu.name;
}
```

同时，也可定义匿名结构体

```GO
myCar := struct {
    Make string
    Model string
} {
    Make: "tesla",
    Model: "model 3"
}

// 匿名结构体嵌套
type Car struct {
    Mkae string
    Model string
    Wheel struct {
        Raidus int
        Material string
    }
}
```

#### 结构体的实例化

Go 结构体的实例化有两种方式：

- 1 使用 `&` 直接初始化
- 2 利用 `new`

```go
s := &Student{
	name: "Tom",
}

s := new(Student)
s.hello("taylor")
```

#### 结构体没有继承？
    
GO 没有继承，但可以使用组合替代

```go
type User struct {
    name string
}

type Student struct {
    grade int
    User
}

stu := &Student {
    name: "Taylor",
    grade: 2,
}
fmt.Println(stu.name)
fmt.Println(stu.grade)
```

### 1.14 接口（interface）

在 Go 语言中，并不需要显式地声明实现了哪一个接口，只需要**直接实现**该接口对应的方法即可

- 没有 `implement` 关键字，实例化成对象后，强制类型转换为接口类型（解耦）
    
```go
// 接口
type People interface {
    getName() string
}

type Student struct {
    name string
    age int
}

// 实现接口方法
func (s *Student) getName() string {
    return s.name
}

v := Student{"taylor", 22}

stu = &v
ftm.Printfln(stu.getName())  // OK
stb = v
ftm.Printfln(stb.getName())  // ERROR: getName 只在 *Student（指针类型）上定义
```

- 接口可以看成是值和类型的元组
    - `(value, type)`
    ```go
    fmt.Printf("(%v, %T)\n", stu, stu)  // (&{taylor 22}, *main.Student)
    ```
- 空接口表示任意类型
    ```go
    // 1. 接受任何类型的参数
    func read(i interface{}) {
        // do
    }
    
    // 2. Map
    m := make(map[string]interface{})
    m["ni"]  = "hello"
    m["hao"] = 22
    m["a"]   = [2]int{2,2}
    ```

### 1.15 类型判断

断言 x 不为 `nil`，并且存储在 x 中的值的类型为 `TYPE`：

```go
t, ok := x.(TYPE)
```

首先，基本类型转换，使用对应类型的转换操作即可

```go
var x int = 10
var y float64 = float64(x) // 将x转为浮点数
var z int32 = int32(y) // 将y转为32位整数
```

其次，类型的判断，使用断言表达式：

- 返回变量的值和是否匹配状态
- 如果类型不匹配，并且没有捕获匹配状态，则将报错

```go
var v interface{} = "hahaha"

s := v.(string)         // s = "hahaha"
s, ok := v.(string)     // s = "hahaha", ok = true
s, ok := v.(float64)    // s = 0, ok = false
s := v.(float64)        // panic
```

还可以利用 `switch case` 判断类型

- `type`：关键字，仅用于 `swatch case`。

```go
switch v := i.(type) {
    case int:
        // do
    case string:
        // do
    default:
        // no match; here v has the same type as i
}
```


## 02 Generics 泛型

### 2.1 泛型函数

单一泛型比较简单，直接在函数后面声明 `[T 类型]`，其中 `T` 为任意名称，`类型` 可以是具体的内建类型，也可以是自定义的 struct。

```go
func add[T any](a, b T) T {
    return a + b
}

func main() {
    add(1, 2) // 3
    add(1.0, 2.2) // 3.2
}
```

多泛型示例：

```go
// map
func MapKeys[K string, V int](m map[K]V) []K {
    // do sth.
}
```

### 2.2 泛型类型及其函数

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func main() {
    intStack := Stack[int]{}
    intStack.push(1)
    intStack.push(2)
    
    strStack := Stack[string]{}
    strStack.push("Hello")
    strStack.push("World")
}
```

## 03 Go Modules

Go Modules 是 Go 语言用于管理依赖关系的官方解决方案。它的目的是简化和改进 Go 语言项目的依赖管理。

在早期，Go 使用 GOPATH 来管理依赖包，但这种方式存在一些限制和不便。Go Modules 解决了这些问题，它的关键点包括：

- 版本管理
- 依赖拉取：使用 `go get` 拉取依赖，不需要依赖于 `GOPATH` 的特定目录结构
- 版本控制
- 代理支持

### 3.1 创建项目

首先，创建一个空目录，然后使用 `go mod init PROJECT` 初始化，生成一个 `go.mod` 文件：

```shell
$ mkdir helloWorld
$ cd helloWorld

$ go mod init hello
$ ls
go.mod

$ cat go.mod                      
module hello

go 1.19.1
```

然后，添加 `main.go`，执行编译命令

```shell
$ go build
$ ls
go.mod  hello  main.go

$ ./hello
Hello World~
```

接着，使用 `go get` 安装依赖，这里以 Gin 为例：

```go
$ go get -u github.com/gin-gonic/gin
go: downloading github.com/bytedance/sonic v1.10.2
go: downloading github.com/ugorji/go/codec v1.2.12
go: downloading github.com/go-playground/validator v9.31.0+incompatible
go: downloading github.com/go-playground/validator/v10 v10.16.0
.....

$ cat go.mod 
module hello

go 1.21.5

require (
        github.com/bytedance/sonic v1.10.2 // indirect
        github.com/chenzhuoyu/base64x v0.0.0-20230717121745-296ad89f973d // indirect
        ......
)
```

然后，导入依赖，修改程序：

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    // Create router.
    r := gin.Default()
    // Bind.
    r.GET("/", func(c *gin.Context) {
        c.String(http.StatusOK, "Hello World~")
    })
    // Run & Serve on 0.0.0.0:8080.
    r.Run(":8080")
}
```

最后，`go build` 编译后启动服务，使用 curl 校验：

```shell
$ curl http://localhost:8080
Hello World~
```

### 3.2 依赖本地项目（replace）

Go Modules 通常都从网上（github.com）拉取依赖，但有时候我们也需要引用到内部其他项目的依赖，需要使用 `replace` 替换依赖模块的路径。

一般来说，依赖本地项目仅开发测试使用。

例如，项目需要依赖 DDD，但是该项目目前只是测试阶段，并未推送到 github，那么可以使用 `replace` 指定该项目在本机的路径

```sh
module example.com/myproject

require (
    github.com/some/DDD v1.2.3
)

replace github.com/some/DDD v1.2.3 => /path/to/local/DDD
```

当然，`replace` 能替换的东西比较多，本质上就是修改依赖指向，如：

```sh
# 1. 替换版本号
replace github.com/some/DDD v1.2.3 => replace github.com/some/DDD v1.1.1
# 2. 换源
replace github.com/some/DDD v1.2.3 => replace mycloud.com/some/DDD v1.1.1
```

## 04 Go Channels & Concurrency

### 4.1 Goroutines 协程

Goroutine 很简单，使用 `go f(x, y, z)` 即可异步执行，Goroutine 运行在同一个地址空间，因此对共享内存的访问必须同步。

```GO
func f(x, y, z int) { //... }

func main() {
    go f(x, y, z)
}
```

### 4.2 Channels 管道/通道

通道是一种类型化的管道，您可以通过它使用通道运算符 `<-` 发送和接收值。

```go
// Channel 必须先创建
ch := make(chan int)

// 向通道发送数据
ch <- v    // Send v to channel ch.

// 从通道读取数据
v := <-ch  // Receive from ch, and
           // assign value to v.
```

- 默认向通道发送和接收数据时阻塞，直到发送/接收完成
- 从未初始化的通道（nil）读取数据，将死锁

### 4.3 Buffer Channels 缓冲通道

通道可以缓冲，创建通道时指定缓冲大小即可：

```go
ch := make(chan int, 100)
```

普通通道默认阻塞，而缓冲通道只会在两种情况下阻塞：
1. 缓冲通道满时，发送阻塞
2. 缓冲通道为空时，接收阻塞

### 4.4 Channel Range & Close

通道可以使用 Range 来遍历接收：

```go
// v, ok := <- ch
// ok is false if ch has closed, or it is true

ch := make(chan int, 10)

// 这里只推送了 5 个，所以需要主动关闭通道
go f(5, ch)

for i := range ch {
    // print
}
```

发送者可以主动关闭通道（接收者不行）， 当然，通道和文件不同，通常不用关闭。

```go
func f(n int, ch chan int) {
    for i := 0; i < n; i++ {
        ch <- i + 1
    }
    // 关闭 ch，以防 x 小于 ch 缓冲长度
    close(ch)
}
```

### 4.5 Select 多通道监听

上述 Channel 和 Buffer Channel 都是针对单个通道的，如果要同时接收多个通道的信息，需要使用 `Select`，它类似 switch 语法，只不过它用于通道监听。

```go
for {
    select {
        case i, ok := <- ch1:
            // do sth.
        case j, ok := <- ch2:
            // do sth.
    }
}
```

当 Select 监听的其中一个通道接收到数据，对执行对应的 case；如果同时多个通道都接收到消息，那么会**随机**选择一个（是不是有点类似 C 的 select、poll、epoll？）

Select 默认是阻塞等待的，但是加上 `default` 之后，它就变成类似 `try lock` 的非阻塞监听，通道有数据时读取，如果没有直接往下执行：

```go
select {
    case i, ok := <- ch
        // do sth.
    default:
        // 从 ch 接收时会阻塞
}
```

### 4.6 只读、只写限制

声明 Channel 时，可以限制它的读写权限：

- `<-chan T`：只读通道
- `chan<- T`：只写通道

```go
func readOnly(ch <-chan int) {
    // ch can only be read from.
}

func wirteOnly(ch chan<- int) {
    // ch can only be write to.
}

func main() {
    ch := make(chan int)
    
    writeOnly(ch)
    readOnly(ch)
}
```

## 05 锁

### 5.1 Mutex 互斥锁

互斥锁（Mutex，全称 Mutual Exclusion）是一种同步机制，用于确保在任意时刻只有一个线程能够访问共享资源，从而避免多个线程同时修改相同的数据造成的问题。

Mutex 有如下两个方法：

- `sync.Lock()`：加锁
- `sync.Unlock()`：解锁

```go
import (
	"sync"
)

var sharedResource int
var mutex sync.Mutex

func inc() {
	// 加锁
	mutex.Lock()
	defer mutex.Unlock() // 在函数结束时释放锁

	// 访问或修改共享资源
	sharedResource++
	// ...
}

func main() {
    // go inc() ...
}
```

### 5.2 RWMutex 读写锁

RWMutex 读写互斥锁，它相比于普通的互斥锁 Mutex 提供了更灵活的读写控制。在 Go 语言中，RWMutex 可以同时允许多个 goroutine 获取读取锁，但在写入锁被获取时，所有的读取和写入操作都会被阻塞。

除了有 Mutex 一样的 `Lock` 和 `Unlock` 方法，还有读写锁：

- `sync.RLock()`：加读锁，多个 goroutine 可同时加锁，不影响读。但如果被 Lock，则 RLock 不能再加锁，将被阻塞
- `sync.RUnlock()`：解锁



## REFERENCES

[1] 官方文档. https://tour.golang.org/.  
[2] 极客兔兔. Go 语言简明教程[DB/OL]. https://geektutu.com/post/quick-golang.html.  
