---
title: Day5.1C++ shared_mutex底层
date: 2026-01-05
tags:
  - Multi-Threading
  - BestPractices
  - "#c-plus-plus"
  - mutex
---
今天研究std::shared_mutex**底层源码**：

```cpp
#if __cplusplus >= 201703L  
  /// The standard shared mutex type.  
  class shared_mutex  
  {  
  public:  
    shared_mutex() = default;  
    ~shared_mutex() = default;  
  
    shared_mutex(const shared_mutex&) = delete;  
    shared_mutex& operator=(const shared_mutex&) = delete;  
  
    // Exclusive ownership  
  
    void lock() { _M_impl.lock(); }  
    [[nodiscard]] bool try_lock() { return _M_impl.try_lock(); }  
    void unlock() { _M_impl.unlock(); }  
  
    // Shared ownership  
  
    void lock_shared() { _M_impl.lock_shared(); }  
    [[nodiscard]] bool try_lock_shared() { return _M_impl.try_lock_shared(); }  
    void unlock_shared() { _M_impl.unlock_shared(); }  
  
#if _GLIBCXX_USE_PTHREAD_RWLOCK_T  
    typedef void* native_handle_type;  
    native_handle_type native_handle() { return _M_impl.native_handle(); }  
  
  private:  
    __shared_mutex_pthread _M_impl;  
#else  
  private:  
    __shared_mutex_cv _M_impl;#endif  
  };  
#endif // C++17
```
可以看到std::shared_mutex **基本上就是__shared_mutex_pthread的一层包装壳**，
另外还有一个非系统API的实现__shared_mutex_cv，但我就不关注这个了。


打开__shared_mutex_pthread可以看到

```cpp
   void  
   lock()  
   {  
     int __ret = __glibcxx_rwlock_wrlock(&_M_rwlock);  
     if (__ret == EDEADLK)  
__throw_system_error(int(errc::resource_deadlock_would_occur));  
     // Errors not handled: EINVAL  
     __glibcxx_assert(__ret == 0);  
   }  
  
   bool  
   try_lock()  
   {  
     int __ret = __glibcxx_rwlock_trywrlock(&_M_rwlock);  
     if (__ret == EBUSY) return false;  
     // Errors not handled: EINVAL  
     __glibcxx_assert(__ret == 0);  
     return true;  
   }  
  
   void  
   unlock()  
   {  
     int __ret __attribute((__unused__)) = __glibcxx_rwlock_unlock(&_M_rwlock);  
     // Errors not handled: EPERM, EBUSY, EINVAL  
     __glibcxx_assert(__ret == 0);  
   }  
  
   // Shared ownership  
  
   void  
   lock_shared()  
   {  
     int __ret;  
     // We retry if we exceeded the maximum number of read locks supported by  
     // the POSIX implementation; this can result in busy-waiting, but this     // is okay based on the current specification of forward progress     // guarantees by the standard.    
      do  
__ret = __glibcxx_rwlock_rdlock(&_M_rwlock);  
     while (__ret == EAGAIN);  
     if (__ret == EDEADLK)  
__throw_system_error(int(errc::resource_deadlock_would_occur));  
     // Errors not handled: EINVAL  
     __glibcxx_assert(__ret == 0);  
   }  
  
   bool  
   try_lock_shared()  
   {  
     int __ret = __glibcxx_rwlock_tryrdlock(&_M_rwlock);  
     // If the maximum number of read locks has been exceeded, we just fail  
     // to acquire the lock.  Unlike for lock(), we are not allowed to throw     // an exception.     if (__ret == EBUSY || __ret == EAGAIN) return false;  
     // Errors not handled: EINVAL  
     __glibcxx_assert(__ret == 0);  
     return true;  
   }  
  
   void  
   unlock_shared()  
   {  
     unlock();  
   }
```

由于是读写锁所以比普通的mutex多了一套对lock_shared的操作。

但我们仔细研究代码，实际上
```cpp
#ifdef PTHREAD_RWLOCK_INITIALIZER  
    pthread_rwlock_t    _M_rwlock = PTHREAD_RWLOCK_INITIALIZER;
```
关键的成员变量 **M_rwlock实际上是一个INT64类型变量**，也就是long long类型，它会初始化为-1。

**它们都调用一套WINDOWS API** 由于WINDOWS不开源使用DDL导入的，所以无从了解底层实现

```cpp
_GLIBCXX_GTHRW(rwlock_rdlock)

_GLIBCXX_GTHRW(rwlock_tryrdlock)

_GLIBCXX_GTHRW(rwlock_wrlock)

_GLIBCXX_GTHRW(rwlock_trywrlock)

_GLIBCXX_GTHRW(rwlock_unlock)
```

我的理解是这个 M_rwlock有64位，系统在内部维护了不同的标志位，执行结束后**根据不同的结果宏定义了它们的错误码。**然后根据返回值判断是否抛出异常，是否继续执行

比如说**读者锁可能是在一部分位置维护了一个读者数量**，**写者锁使用bit位维护是否上锁**。

它的逻辑是:当写者锁上锁后，**不再允许读者增加**，此后进入的读者线程都会阻塞挂起，
但是也**不会立刻开始写入**，需要阻塞挂起读者完全离去即读者数量**等于0时才会开始写入**。

比较有意思的是
```cpp
   lock_shared()  
   {  
     int __ret;  
     // We retry if we exceeded the maximum number of read locks supported by  
     // the POSIX implementation; this can result in busy-waiting, but this     // is okay based on the current specification of forward progress     // guarantees by the standard.    
      do  
__ret = __glibcxx_rwlock_rdlock(&_M_rwlock);  
     while (__ret == EAGAIN);  
     if (__ret == EDEADLK)  
__throw_system_error(int(errc::resource_deadlock_would_occur));  
     // Errors not handled: EINVAL  
     __glibcxx_assert(__ret == 0);  
   }  
```

它先通过**返回值判断读者锁是否溢出**（因为分配的位数是有限的），溢出后会进行循环请求。
也就是说等到**其他读者释放了锁它才能拿到锁**。

以上就是全部内容，是个**比较有意思的实现**，可能是通过原子操作同一个变量来保证安全高效。