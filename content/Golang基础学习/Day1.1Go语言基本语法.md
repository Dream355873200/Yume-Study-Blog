---
title: Day1.1Go语言变量声明
date: 2026-01-21
tags:
  - Golang
---
## 1.前言

**C++编写高性能网络服务器还是不够高效**
在需要极致高性能，无GC抖动**才会使用c++进行开发网络服务器**，比如金融，渲染等。

所以我想要**学习Golang作为一门新的后端语言**进行开发。

当然我也学过springboot，但是我觉得**JVM过于重了**，开发web**又要装tomcat，maven**。
java的框架和语法等等还是没有Go语言优雅。
**而且Go语言层面就支持并发操作，一个go就能启动一个轻量协程，GMP又能高性能调度。**

所以今天开始学习Golang

### 2.golang基本语法

**go语言的几种变量声明方式**
```go
var c int  
var b=0  
var(  
    d int  
    e string  
)
f := 3.1415
```
首先是**var变量声明，它可以在全局，函数块内进行声明**
可以**自动推断变量类型**
也可以**类型后置标明类型**。
然后**短变量声明 f:=3.1415 不能在全局作用域使用**，它主要是进行<font color="#f79646">便捷声明变量</font>，可以自动推断类型。

注意：局部变量不使用是无法通过编译的，而全局变量可以

go语言的常量声明
```go
const h = 1
const (  
    a=iota  
    b  
    c 
    d 
    e)
```
上面列举了go语言的const变量声明和const的枚举应用
iota是初始值为0，它会在下一行+1，也就是说第一行赋值的a=0，然后逐行iota会++
也就是b=1 c=2 d=3 e=4
> [!note]
> `iota` 是 Go 语言的一个**常量计数器**。它的核心特点是：**在 `const` 关键字出现时将被重置为 0，并在常量组中每新增一行，计数就会自动加 1，并且只能在const使用。**

go语言也**有我认为不好的地方**，那就是它的{}和()表达式只支持这样：

```go
func test(){

}//正确

func test()
{

}//错误
```
它的编译器规定了只能这么写，不能把第一个大括号换行到下一行，导致代码结构不太清晰。

还有它的**if与for在写条件或循环条件是没有括号**的，比如：
```go

if a>0{

}

for i := 0; i < 10; i++ {

fmt.Println(i)
}

for index, value := range nums {

fmt.Printf("索引：%d, 值：%d\n", i, v)

}
```
而且需要注意的是**它没有while循环关键字**

这是go的switch写法：

```go
finger := 3

switch finger {

case 1:

fmt.Println("大拇指")

case 2:

fmt.Println("食指")

case 3:

fmt.Println("中指")

default: // 所有 case 都不匹配时执行

fmt.Println("无效的手指")

}
```
我认为比较好的是，它**终于不再需要每个case都要写break来跳出了**，我认为c/c++的这一点是比较反人类的。
同时go的**case里的条件支持很多类型**而且支持变量，比c只能写数值或char好多了。

**go语言的包导入方式**
```go
import "fmt"
import (  
    "fmt"  
    "time")
    

```
单行导入就直接使用import 包名，进行导入，多行就直接import ()

与其他语言一样可以有**导包别名**，只需要在import后写一个别名就好了
```go
import (
f "fmt"
r "math/rand"
)

func main() {
f.Println(r.Intn(100)) // 使用 f 和 r
}

```

go语言编译运行指令：go run main.go 会自动编译执行。
然后 go build是执行构建命令会打包出可执行二进制文件，可以直接./运行

