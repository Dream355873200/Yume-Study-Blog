---
title: Day6.1C++ shared_ptr底层
date: 2026-01-06
tags:
  - Multi-Threading
  - BestPractices
  - "#c-plus-plus"
  - shared_ptr
---
今天学习`std::shared_ptr`**底层源码**

```cpp
template<typename _Tp>  
  class shared_ptr : public __shared_ptr<_Tp>
```

可以看出`std::shared_ptr`和昨天的`shared_mutex`**一样**对`__shared_ptr`封装了一层调用。

那我们来看`__shared_ptr`：
```cpp
private:
element_type*      _M_ptr;         // Contained pointer.  
__shared_count<_Lp>  _M_refcount;    // Reference counter.
```
它有且只有**两个重要的成员变量**，`_M_ptr`存储指针，`_M_refcount`存储计时器。

发现`__shared_ptr`的**重要的构造函数和析构函数**：

```cpp
  constexpr __shared_ptr() noexcept  
     : _M_ptr(0), _M_refcount()  
     { }  
  
     template<typename _Yp, typename = _SafeConv<_Yp>>  
explicit  
__shared_ptr(_Yp* __p)  
: _M_ptr(__p), _M_refcount(__p, typename is_array<_Tp>::type())  
{  
  static_assert( !is_void<_Yp>::value, "incomplete type" );  
  static_assert( sizeof(_Yp) > 0, "incomplete type" );  
  _M_enable_shared_from_this_with(__p);  
}  
  
     template<typename _Yp, typename _Deleter, typename = _SafeConv<_Yp>>  
__shared_ptr(_Yp* __p, _Deleter __d)  
: _M_ptr(__p), _M_refcount(__p, std::move(__d))  
{  
  static_assert(__is_invocable<_Deleter&, _Yp*&>::value,  
      "deleter expression d(p) is well-formed");  
  _M_enable_shared_from_this_with(__p);  
}  
  
     template<typename _Yp, typename _Deleter, typename _Alloc,  
       typename = _SafeConv<_Yp>>  
__shared_ptr(_Yp* __p, _Deleter __d, _Alloc __a)  
: _M_ptr(__p), _M_refcount(__p, std::move(__d), std::move(__a))  
{  
  static_assert(__is_invocable<_Deleter&, _Yp*&>::value,  
      "deleter expression d(p) is well-formed");  
  _M_enable_shared_from_this_with(__p);  
}
```

同时**全是defalut的拷贝和析构**：
```cpp
__shared_ptr(const __shared_ptr&) noexcept = default;  
__shared_ptr& operator=(const __shared_ptr&) noexcept = default;  
~__shared_ptr() = default;
```

也就是说：**拷贝是直接拷贝指针和couter对象的。**

而`__shared_ptr`的析构函数是defalut所以没对资源指针处理
构造函数也把资源指针传入了couter。

**所以`std::shared_ptr`主要逻辑其实是在couter这个类中的。**

我们来看couter：
```cpp
__shared_count(_Ptr __p) : _M_pi(0)  
{  
  __try  
    {  
      _M_pi = new _Sp_counted_ptr<_Ptr, _Lp>(__p);  
    }  
  __catch(...)  
    {  
      delete __p;  
      __throw_exception_again;  
    }  
}
```

研究发现它是在第一个shared_ptr对象创建时使用的，构造了一个计数器基类,`_Sp_counted_base<_Lp>*  _M_pi;`


```cpp
__shared_count(_Ptr __p, _Deleter __d)  
: __shared_count(__p, std::move(__d), allocator<void>())  
{ }  
  
     template<typename _Ptr, typename _Deleter, typename _Alloc,  
       typename = typename __not_alloc_shared_tag<_Deleter>::type>  
__shared_count(_Ptr __p, _Deleter __d, _Alloc __a) : _M_pi(0)  
{  
  typedef _Sp_counted_deleter<_Ptr, _Deleter, _Alloc, _Lp> _Sp_cd_type;  
  __try  
    {  
      typename _Sp_cd_type::__allocator_type __a2(__a);  
      auto __guard = std::__allocate_guarded(__a2);  
      _Sp_cd_type* __mem = __guard.get();  
      ::new (__mem) _Sp_cd_type(__p, std::move(__d), std::move(__a));  
      _M_pi = __mem;  
      __guard = nullptr;  
    }  
  __catch(...)  
    {  
      __d(__p); // Call _Deleter on __p.  
      __throw_exception_again;  
    }  
}
```
然后传入删除器 `_Deleter`实际上会将删除器移动到m_pi中。

所以我们神奇的发现！**以上的所有类实际上都是对`_Sp_counted_base<_Lp>*  _M_pi;`的一个封装**

也就是说，`_Sp_counted_base`和派生类才是**最核心**的，由它来**存储计数变量**，由它来**析构new的资源**，由它来**执行`_Deleter`**。

我们最终来看`_Sp_counted_base`：
```cpp
private:  
  _Sp_counted_base(_Sp_counted_base const&) = delete;  
  _Sp_counted_base& operator=(_Sp_counted_base const&) = delete;  
  
  _Atomic_word  _M_use_count;     // #shared  
  _Atomic_word  _M_weak_count;    // #weak + (#shared != 0)
```
它只存储了计数变量int类型，然后我们发现**它是有一些虚函数**
```cpp
virtual  
~_Sp_counted_base() noexcept  
{ }  
  
// Called when _M_use_count drops to zero, to release the resources  
// managed by *this.  
virtual void  
_M_dispose() noexcept = 0;  
  
// Called when _M_weak_count drops to zero.  
virtual void  
_M_destroy() noexcept  
{ delete this; }  
  
virtual void*  
_M_get_deleter(const std::type_info&) noexcept = 0;
```



于是找到了**继承类**

```cpp
class _Sp_counted_ptr final : public _Sp_counted_base<_Lp>  
{  
public:  
  explicit  
  _Sp_counted_ptr(_Ptr __p) noexcept  
  : _M_ptr(__p) { }  
  
  virtual void  
  _M_dispose() noexcept  
  { delete _M_ptr; }  
  
  virtual void  
  _M_destroy() noexcept  
  { delete this; }  
  
  virtual void*  
  _M_get_deleter(const std::type_info&) noexcept  
  { return nullptr; }  
  
  _Sp_counted_ptr(const _Sp_counted_ptr&) = delete;  
  _Sp_counted_ptr& operator=(const _Sp_counted_ptr&) = delete;  
  
private:  
  _Ptr             _M_ptr;  
};
```

它是 `_Sp_counted_base`的**继承类**，所以它有**计数变量`_M_use_count`和资源指针 `_M_ptr`**


当计数变量减少时，**先判断是否为1**(最后的shared_ptr)
```cpp
_Sp_counted_base<_S_mutex>::_M_release() noexcept  
   {  
     // Be race-detector-friendly.  For more info see bits/c++config.  
     _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_use_count);  
     if (__gnu_cxx::__exchange_and_add_dispatch(&_M_use_count, -1) == 1)  
{  
  _M_release_last_use();  
}  
   }
```

**调用 last_use**
```cpp
 _M_release_last_use() noexcept  
     {  
_GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_use_count);  
_M_dispose();  
// There must be a memory barrier between dispose() and destroy()  
// to ensure that the effects of dispose() are observed in the  
// thread that runs destroy().  
// See http://gcc.gnu.org/ml/libstdc++/2005-11/msg00136.html  
if (_Mutex_base<_Lp>::_S_need_barriers)  
  {  
    __atomic_thread_fence (__ATOMIC_ACQ_REL);  
  }  
  
// Be race-detector-friendly.  For more info see bits/c++config.  
_GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_weak_count);  
if (__gnu_cxx::__exchange_and_add_dispatch(&_M_weak_count,  
                   -1) == 1)  
  {  
    _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_weak_count);  
    _M_destroy();  
  }  
     }
```

然后调用`_M_dispose()`;  进行**资源的释放**也就是delete M_Ptr;
```cpp
_M_dispose() noexcept  
{ delete _M_ptr; }
```


然后**弱引用M_weak_count计数为0时才会删除这个计数器本身**，`_M_destroy();` `delete this;`

这个是**普通派生类**，还有一个**删除器版本的派生**，会调用**删除器函数来析构**

分别对应：

普通派生的`__shared_count`构造
```cpp
__shared_count(_Ptr __p) : _M_pi(0)  
{  
  __try  
    {  
      _M_pi = new _Sp_counted_ptr<_Ptr, _Lp>(__p);  
    }  
  __catch(...)  
    {  
      delete __p;  
      __throw_exception_again;  
    }  
}
```

删除器派生的`__shared_count`构造
```cpp
   template<typename _Ptr, typename _Deleter, typename _Alloc,  
       typename = typename __not_alloc_shared_tag<_Deleter>::type>  
__shared_count(_Ptr __p, _Deleter __d, _Alloc __a) : _M_pi(0)  
{  
  typedef _Sp_counted_deleter<_Ptr, _Deleter, _Alloc, _Lp> _Sp_cd_type;  
  __try  
    {  
      typename _Sp_cd_type::__allocator_type __a2(__a);  
      auto __guard = std::__allocate_guarded(__a2);  
      _Sp_cd_type* __mem = __guard.get();  
      ::new (__mem) _Sp_cd_type(__p, std::move(__d), std::move(__a));  
      _M_pi = __mem;  
      __guard = nullptr;  
    }  
  __catch(...)  
    {  
      __d(__p); // Call _Deleter on __p.  
      __throw_exception_again;  
    }  
}
```


其实**添加计数**挺简单的：

```cpp
 __shared_count(const __shared_count& __r) noexcept  
     : _M_pi(__r._M_pi)  
     {  
if (_M_pi != nullptr)  
  _M_pi->_M_add_ref_copy();  
     }  
  
     __shared_count&  
     operator=(const __shared_count& __r) noexcept  
     {  
_Sp_counted_base<_Lp>* __tmp = __r._M_pi;  
if (__tmp != _M_pi)  
  {  
    if (__tmp != nullptr)  
      __tmp->_M_add_ref_copy();  
    if (_M_pi != nullptr)  
      _M_pi->_M_release();  
    _M_pi = __tmp;  
  }  
return *this;  
     }
```

```cpp
void  
_M_add_ref_copy()  
{ __gnu_cxx::__atomic_add_dispatch(&_M_use_count, 1); }
```

就是**拷贝赋值**和**拷贝构造**的时候让计数器+1。


好的以上就是`std::shared_ptr`的**核心内容**，核心逻辑**藏得很深**，其他都是对**易用性进行的封装**。