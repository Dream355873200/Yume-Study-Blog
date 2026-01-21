---
title: Day1.2Go函数多返回值
date: 2026-01-21
tags:
  - Golang
---

本章学习Go语言函数多返回值

go语言的函数定义关键字是`func`
一个基本函数定义如下
```go
func stringadd(a string, b string) string {  
    c := a + " " + b  
    return c  
}
```
它是基本的字符串拼接，展示基本的函数定义，依然是变量类型后置，以及与C++不同的是参数后还有一个返回值类型参数。

多返回值版本：
```go
func string_echo(a string, b string) (string,string) {  
    return a,b  
}
```
go可以定义多返回值函数，它还能是这样：
```go
func string_echo_1(a string, b string) (c string, d string) {  
    c=a  
    d=b  
    return c,d  
}
```
或者这样：
```go
func string_echo_1(a string, b string) (c string, d string) {  
    return a,b  
}
```
返回值形参主要就是相当于在函数内直接定义变量
特别的就是如果直接写 return，它会默认返回返回值形参。