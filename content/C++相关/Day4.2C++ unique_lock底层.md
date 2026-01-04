---
title: Day4.2C++ unique_lock底层
date: 2026-01-01
tags:
  - Multi-Threading
  - BestPractices
  - Initialize
  - c-plus-plus
  - mutex
  - Dead-Lock
---
今天学习std::unique_lock的底层原理

研究源码发现它有且只有**两个关键的私有变量**

```cpp
private:  
  mutex_type*   _M_device;  //typedef _Mutex mutex_type;
  bool     _M_owns;
```
这里是它的构造函数

```cpp

unique_lock() noexcept  
: _M_device(0), _M_owns(false)  
{ }

 explicit unique_lock(mutex_type& __m)  
     : _M_device(std::__addressof(__m)), _M_owns(false)  
     {  
lock();  
_M_owns = true;  
     }  
  
     unique_lock(mutex_type& __m, defer_lock_t) noexcept  
     : _M_device(std::__addressof(__m)), _M_owns(false)  
     { }  
  
     unique_lock(mutex_type& __m, try_to_lock_t)  
     : _M_device(std::__addressof(__m)), _M_owns(_M_device->try_lock())  
     { }  
  
     unique_lock(mutex_type& __m, adopt_lock_t) noexcept  
     : _M_device(std::__addressof(__m)), _M_owns(true)  
     {  
// XXX calling thread owns mutex  
     }
~unique_lock()  
     {  
      if (_M_owns)  
       unlock();  
     }
```
通过这一串构造函数不难发现
其中  **M_device存储了各种构造传入的互斥量指针**，**M_owns存储了锁的状态**。

由于有锁的状态，所以我们可以调用owns_lock判断是否已经加锁(**假如你正确执行加解锁**)

上面有三个基本的构造：（**未包含全部**）

defer是把M_own设置为false、
adopt是把M_own设置成true、

仅传入互斥量的构造函数会**主动加锁**。
try_to_lock是尝试加锁，所以**它可以在你不确定是否已经上锁时使用**（不会引发异常）

比较安全的是 unique_lock在**析构前先会判断M_own**，再进行解锁，
而lock_gurad的析构**只有一行unlock**。

它的特点是：**能够主动调用lock和unlock进行加锁，析构时会自动解锁**。

另外重要的是，
无论是lock_gurad还是unique_lock的adopt，都是**默认你的互斥量已经上锁的**。
unique_lock的defer，**默认你的互斥量没有上锁**。

这点要注意，在lockgurad上领养一个未加锁的互斥量是会崩溃的，因为**重复解锁行为是Undefined**。

所以我们可以发现，**unique_lock基本可以认为是lock_gurad的一个更安全的升级版**（但也更重）。
