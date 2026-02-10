---
title: Day2.1Go的Map与接口结构体
date: 2026-01-22
tags:
  - Multi-Threading
  - BestPractices
  - "#c-plus-plus"
---
今天学习Go的Map、interface、struct

## 1.Map相关

Go的**map实际上是哈希Map**，**std::unordered_map**

它的定义如下
```go

map0 := make(map[string]int)  
  
var map1 map[string]int  
  
map2:= map[string]string{  
    "123": "123",  
    "456": "456",  
}
```
需要注意的是：**Map是一定要在使用前初始化的**，比如第二种
`var map1 map[string]int`它实际上只给它赋值了一个nil，
需要在使用前调用make(map[]type)

`map` 变量本质上是一个 **`*hmap` 指针**，所以在底层上需要先初始化，不然是nil

所以当map作为参数时可以直接访问原map数据
map在存储不够后会自动扩容

与slice扩容不同的是，slice存储的是结构体本身，**扩容会创建一个新结构体，将数据整体搬迁**，所有当slice作为函数参数时，外部扩容后函数参数的修改实际上修改的是**搬迁前的数据**。
而map存储的是**hmap结构体的指针**，所以扩容后hmap内部的资源指针改变了，但map变量不变。

```go
for key,value:= range map{

}
```
map的**range遍历实际上会分别取出来key和value**。
复制出来的也是副本，不会修改原数据，除非你的k/v是指针。

因为它实际上是一个指针所以赋值实际上赋的是指针

删除数据可以调用`delete(map,key)`



## 2.Struct相关

struct是go的结构体，基本定义如下
```go
type book struct {  
    name string  
    price float32  
}
```
与函数同样，结构体首字母和变量首字母决定了可见性。

Go独特的是有接收者
分别是指针接收者和值接收者
区别就是
```go
func (c *Cat) Sleep() {  
    fmt.Println("猫睡觉")  
}
func (c Cat) Sleep() {  
    fmt.Println("猫睡觉")  
}
```
一个是指针类型，一个是值类型。
定义就是在函数名前写一个参数，根据变量类型分配到指定的类型
Go 语言规定：**接收者的类型定义必须和方法定义在同一个包（package）中**。

另外type是一个强大的关键字，它可以定义一个新类型，比如说上面的book类型
甚至可以给int定义新名
```go
type Myint int
```
注意：它可不是c/c++简单的typedef，只是起个别名，Go的type是真正意义定义了新类型。
比如int不能直接赋值给Myint类型，而且你可以给Myint配置接收者函数


结构体的组合是这样的：

```go
type human struct {  
    name string  
    sex  string  
}  
  
type student struct {  
    human  
    age     int  
    college string  
}
```

student可以直接使用.运算符访问name与sex
但是直接初始化需要这样
```go
s:=student{  
    human:   human{name:"",sex:""},  
    age:     0,  
    college: "",  
}
```
不过当**组合了多个拥有同名属性的子结构体**时需要`.子结构体.属性`

同样，你可以写一个与子结构体同名的方法，可以直接.运算符调用，也可以通过`.子结构体.方法`调用原方法。
## 3.interface与多态

interface同样需要type来定义
```go
type animal interface {  
    Sleep()  
    Eat()  
}  
```
同样，接口也可以进行组合，通过多个小接口组合成一个大的。
```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }

// 组合成一个新的读写接口
type ReadWriter interface {
    Reader
    Writer
}
```

与其他语言不同的是，Go的接口不能写变量，它只能定义方法。



Go的多态是通过interface实现的，比如:
```go
  
type animal interface {  
    Sleep()  
    Eat()  
}  
  
type Cat struct {  
}  
  
func (c *Cat) Eat() {  
    //TODO implement me  
    panic("implement me")  
}  
  
func (c *Cat) Sleep() {  
    fmt.Println("猫睡觉")  
}  
  
func main() {  
  
    cat := &Cat{}  
    var a animal  
    a = cat  
    a.Sleep()  
    cat.Sleep()  
}
```
    a.Sleep()  
    cat.Sleep()  会执行同一个函数


Go遵循鸭子模型：“如果它走起路来像鸭子，叫起来也像鸭子，那么它就是鸭子。” _(If it walks like a duck and quacks like a duck, then it must be a duck.)_

它的接口不需要显式继承，而是当你给结构体实现了这个接口，那么它就可以使用多态。
它优雅的是：**依赖关系的极致解耦**。

而且有一个空接口interface{}，那就是所有结构体都实现了这个接口，那么所有结构体都可以赋值给interface{}。

在go新版本中有一个any关键字代指interface{}，它就像java的object，但是**又比它轻量**。
