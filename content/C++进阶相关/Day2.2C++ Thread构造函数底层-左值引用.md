---
title: Day2.2C++ Thread构造函数底层-左值引用
date: 2026-01-02
tags:
  - Multi-Threading
  - "#c-plus-plus"
  - Left-Reference
  - thread
---

## 1. Thread的函数调用检查

假如传入Thread的函数参数是一个非const的左值引用类型，例如：

```cpp
void changeparam(int &param)  
{  
    param++;  
}  
void test(int someparm)  
{  
  
    std::thread t1(changeparam,someparm);  //此处会报错
    t1.join();  
    std::cout<<someparm;  
}
```
正常调用会触发一个static_assert的**函数调用检查错误**

因为thread的构造函数是这么写的：

```cpp
 thread(_Callable&& __f, _Args&&... __args)  
     {  
static_assert( __is_invocable<typename decay<_Callable>::type,  
                typename decay<_Args>::type...>::value,  
  "std::thread arguments must be invocable after conversion to rvalues"  
  );  
  
using _Wrapper = _Call_wrapper<_Callable, _Args...>;  
// Create a call wrapper with DECAY_COPY(__f) as its target object  
// and DECAY_COPY(__args)... as its bound argument entities.  
_M_start_thread(_State_ptr(new _State_impl<_Wrapper>(  
      std::forward<_Callable>(__f), std::forward<_Args>(__args)...)),  
    _M_thread_deps_never_run);  
     }
```
其中
```cpp
static_assert( __is_invocable<typename decay<_Callable>::type,  
                typename decay<_Args>::type...>::value,  
  "std::thread arguments must be invocable after conversion to rvalues"  
  );  
```
这段代码在调用static_assert之前**会先进行一次decay调用**





> [!note]
> std::decay的作用是移除引用和const volatile限定符，并将数组/函数转换为指针，返回一个类型

> [!note]
> `std::declval<T>()` 的返回值取决于你传入的模板参数 `T`：
> 
> - **如果 `T` 是引用类型**：它返回 `T` 本身。
>     
> - **如果 `T` 不是引用类型**：它返回 `T&&`（右值引用）。
>     
> 
> > **注意：** `std::declval` **没有定义（Implementation）**，只有声明。这意味着你**不能在运行期调用它**。如果你尝试在代码中真的执行它，编译器会报错。它只能出现在 `decltype`、`sizeof` 等不求值语境（Unevaluated context）中。

所以当传入任何类型时，经过万能引用会变成一个引用类型，
在经过decay时会移除引用产出了一个**普通类型**，但 `__is_invocable` 在进行模拟调用时， `__is_invocable`内部又调用了一次 **std::declval**将其转换为右值引用

也就是说这行代码，**它的意思是：固定的把任意类型转换为右值引用，因为右值引用只有在非const左值引用参数绑定才通不过**

std::decay会**将各种引用去除变成普通类型**

std::declval会**把一个普通类型转换为一个右值引用类型**


**右值引用当然无法绑定到一个非const的左值引用上**

引发报错：`error: static assertion failed: std::thread arguments must be invocable after conversion to rvalues`

**我认为c++标准这么写是有好处的：**
因为在传入一个普通类型时，**内部会将其转换为临时tuple存储起来**，假如将这个临时tuple绑定到非const左值引用上毫无意义，因为是非const左值引用，修改一个临时变量毫无意义，引发没有必要的精力去检查，所以直接不让程序员这么做。


假如不这么做，**其实也会引发报错**，因为
```cpp
_M_invoke(_Index_tuple<_Ind...>)  
{ return std::__invoke(std::get<_Ind>(std::move(_M_t))...); }
```
在函数调用时会触发invoke进行std::move强制转换为右值引用类型，
右值引用当然不能绑定到一个非const左值引用上，这样是在**运行时报错**信息就难以寻找了。
所以**结论是这个类型检查可以让报错拦截在编译期，更好追踪、更规范**

<mark style="background:rgba(240, 200, 0, 0.2)">但是有时候我们真的想要去绑定一个普通变量到非const左值引用上</mark>，这时候我们需要使用std::ref
```cpp
void changeparam(int &param)  
{  
    param++;  
}  
void test(int someparm)  
{  
  
    std::thread t1(changeparam,std::ref(someparm));  
    t1.join();  
    std::cout<<someparm;  
}
```
这下就可以正常运行了，因为std::ref实际上是存储了数据的指针，无论内部怎么转换，在调用invoke时使用std::move传递给函数时会进行隐式转换变成原变量的引用。

**但是这么做是很危险的，必须完全掌控它的生命周期，才能避免调用一个已经被释放的空引用引发崩溃**

以下是它的完整流程，**更透彻地理解为什么它能“骗过” `std::thread` 的检查：**

### 1. `decay` 对它的影响

这是最精妙的地方。当使用 `std::ref(someparm)` 时：

- `_Args` 被推导为 `std::reference_wrapper<int>`。
    
- `std::decay` 作用于它时，发现它既不是数组也不是函数，也没有引用符号（它是一个普通的类对象），所以 **`decay` 之后结果还是 `std::reference_wrapper<int>`**。
    
- 这绕过了“变成 `int` 临时变量”的陷阱。
    

### 2. “存储指针”与“隐式转换”

因为`std::declval`的原因，`std::reference_wrapper`也会变成一个右值
但是`std::reference_wrapper` 内部包装了一个 `T*`。

让它能跑通 `static_assert` 的关键在于它的 **`operator T& ()` 重载**：

```cpp
// 简化版示意代码
template<typename T>
class reference_wrapper {
    T* _ptr;
public:
    operator T& () const noexcept { return *_ptr; } // 返回左值引用
};
```

当 `__is_invocable` 模拟调用 `changeparam(wrapper)` 时，由于需要传递引用类型，编译器发现 `wrapper` 可以通过这个**隐式转换的运算符重载**变成 `int&`，正好匹配 `changeparam` 的参数要求。所以 `static_assert` 顺利通过，之后的std::move也同理
