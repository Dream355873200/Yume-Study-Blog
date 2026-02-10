---
title: Day1.2Go函数多返回值
date: 2026-01-21
tags:
  - Golang
---

本章学习**Go语言函数多返回值**

go语言的函数定义关键字是`func`
一个基本函数定义如下
```go
func stringadd(a string, b string) string {  
    c := a + " " + b  
    return c  
}
```
它功能是**基本的字符串拼接**，展示基本的函数定义，依然是变量类型后置
以及与C++不同的是**参数后还有一个返回值类型参数**。

**多返回值版本**：
```go
func string_echo(a string, b string) (string,string) {  
    return a,b  
}
```
go可以定义**多返回值函数**，它还能是这样：
```go
func string_echo_1(a string, b string) (c string, d string) {  
    c=a  
    d=b  
    return c,d  
}
```
或者这样：
```go
func namedReturnDemo() (count int, total string) {
    // 此时 count 是 0, total 是 ""
    // 你可以不赋值直接 return
    return // 返回 0, ""
}
```
返回值形参我认为，主要就相当于在**函数内直接定义变量**，然后直接返回它们。

特别的就是如果**定义了多返回值参数直接函数内写 return**，它会默认返回返回值形参。
而由于go默认会初始化变量为零值或空字符串，这个函数实际上是有返回值的。

由于go没有try-catch，多返回值**又常常再第二个或最后一个参数写的是error来判断是否有错误**。