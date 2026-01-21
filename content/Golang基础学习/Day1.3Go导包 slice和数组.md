---
title: Day1.3Go导包 slice和数组
date: 2026-01-21
tags:
  - Golang
---
本章深入学习go的包机制，以及数组与slice

go语言是递归执行每个包的import的，比如main函数导入lib1，lib1导入lib2。
是先执行lib1的init，然后lib2的init。

这里要介绍一下package的机制

导入packge首先会进行var，const等全局或常量的初始化
如果写了func init(){}，会自动执行init函数，这里就相当于c++的构造函数，可以进行一些RAII或者资源初始化比如数据库注册。

关于包的私有与公有，是通过func的首字母来区分的
比如 func Test()是公有函数，可以通过lib1.Test()调用，如果不是大写就无法外部调用
同样：

```go
func Test()  {  
      
}  
func test()  {  
      
}  
func _Test(){  
      
}
```
最后一个_Test也是私有的，访问类型只根据第一个字母区分。

包导入可以使用别名导入
`import aaa "awesomeProject2/lib1"`
也可以像这样
`import . "awesomeProject2/lib1"`
它会让包内函数可以直接使用函数名调用比如lib1.Test()变成Test()
但是这样可能引发冲突，比如c++的using namespace导致的冲突。

同样如果不想使用包，但是想调用init初始化，可以这样
`import _ "awesomeProject2/lib1"`
它只会调用init，但是不能使用包

注意：以上都不会影响首字母的公有与私有的可见性



## 2.数组与Slice


首先需要理解的是，Go语言的规则是一切都是值传递，不像c++有引用类型，在Go中没有引用！

数组与Slice定义如下
```go
arr:=[2]{1,2}
slice:= []int{}
```
注意：虽然它们很像但却是截然不同的两种数据类型
数组存储的是固定长度的连续空间，直接存储的值。
而slice存储的其实有指针，长度，容量。

所以当作为参数时，传入数组与slice其实都是值拷贝，不同的是指针拷贝可以访问原数据。

可以将arr转换为slice，比如：
```go
slice := arr[:]
```
它是在原数据上进行操作的，不是值拷贝。
遍历数组一般使用slice进行遍历，性能好一些。

Slice可以进行切片
```go
slice := arr[:]//指的是从头到尾全部截取
slice := arr[1:len(arr)]指的是1到末尾
slice := arr[1:4]//指的是1到3索引
```
slice的切片和python类似都是左闭右开，如果想取到末尾可以 arr[n:]直接置空或len(arr)，但是不能超过长度，不然会报错

与c++不同的是，go有一个新概念：容量
比如定义时可以这样
```go
var s = make([]int, 1, 3)  // 1为长度，2为预留容量
```
定义时，会开辟一片3长度的空间，然后初始化长度为默认值。
它们都有有如下方法
cap()返回容量，len()返回长度，copy复制（开辟新空间）


## 3.defer使用

defer是定义语句在函数末尾执行的关键字，类似于栈的执行
它的执行比return要晚一些
```go
func test(){  
      
    defer fmt.Println("test1")  
    defer fmt.Println("test2")  
}
```
它的执行顺序首先是输出test2，然后输出test1，类似栈的执行方式，先入后出
同样可以用来处理RAII相关。