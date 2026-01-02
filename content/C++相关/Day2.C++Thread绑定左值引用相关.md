---
title: C++ 并发编程：Thread的一些问题
date: 2026-01-01
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
会触发一个static_assert的**函数调用检查错误**

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
> std::decay的作用是移除引用和const volatile限定符，并将数组/函数转换为指针

所以当传入非引用类型时，经过万能引用会变成一个左值引用，
在经过decay时触发一次拷贝变成非引用类型，**于是这个拷贝出来的变量就成了临时变量，无法绑定到函数的非const的左值引用类型。**

引发报错：`error: static assertion failed: std::thread arguments must be invocable after conversion to rvalues`

**我认为c++标准这么写是有好处的：**
因为在传入一个普通类型时，**内部会将其转换为临时tuple存储起来**，假如将这个临时tuple绑定到左值引用上毫无意义，因为是非const左值引用，修改一个临时变量毫无意义，引发没有必要的精力去检查，所以直接不让程序员这么做，**除非他们知道函数参数是const类型的左值引用**

假如不这么做，**其实也会引发报错**，因为
```cpp
_M_invoke(_Index_tuple<_Ind...>)  
{ return std::__invoke(std::get<_Ind>(std::move(_M_t))...); }
```
在函数调用时会触发invoke进行std::move强制转换为右值引用类型，
右值引用当然不能绑定到一个非const左值引用上，这样报错信息就难以寻找了。
所以**结论是这个类型检查可以让报错信息更好追踪**

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
这下就可以正常运行了，因为std::ref实际上是存储了数据的指针，无论内部怎么型，在调用invoke时使用std::move传递给函数时会进行隐式转换变成原变量的引用。

**但是这么做是很危险的，必须完全掌控它的生命周期，避免调用一个已经被释放的空引用引发崩溃**

以下是它的完整流程，**更透彻地理解为什么它能“骗过” `std::thread` 的检查：**

### 1. `decay` 对它的影响

这是最精妙的地方。当使用 `std::ref(someparm)` 时：

- `_Args` 被推导为 `std::reference_wrapper<int>`。
    
- `std::decay` 作用于它时，发现它既不是数组也不是函数，也没有引用符号（它是一个普通的类对象），所以 **`decay` 之后结果还是 `std::reference_wrapper<int>`**。
    
- 这绕过了“变成 `int` 临时变量”的陷阱。
    

### 2. “存储指针”与“隐式转换”

`std::reference_wrapper` 内部确实包装了一个 `T*`。

让它能跑通 `static_assert` 的关键在于它的 **`operator T& ()` 重载**：

```cpp
// 简化版示意代码
template<typename T>
class reference_wrapper {
    T* _ptr;
public:
    operator T& () const noexcept { return *_ptr; } // 隐式转换回左值引用
};
```

当 `__is_invocable` 模拟调用 `changeparam(wrapper)` 时，编译器发现 `wrapper` 可以通过这个转换符变成 `int&`，正好匹配 `changeparam` 的参数要求。所以 `static_assert` 顺利通过。

### 3. 为什么 `std::move` 没有错误？

你提到“在 `std::move` 给函数时会进行隐式转换”，这里的细节是：

1. `std::thread` 确实会 `move` 那个 `wrapper` 对象。
    
2. 但是，**移动一个包装引用的对象，并不会移动底层的数据**。它只是把那个内部指针从旧的 `wrapper` 复制到了新的 `wrapper`（在 `std::thread` 的内部存储结构中）。
    
3. 最终调用时，`invoke(wrapper_from_tuple)` 会触发转换，拿到原本那个 `someparm` 的引用。