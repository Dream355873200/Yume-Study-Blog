---
title: Day3.2C++死锁避免
date: 2026-01-03
tags:
  - Multi-Threading
  - BestPractices
  - "#c-plus-plus"
  - Dead-Lock
  - scoped_lock
---

虽然我们想：我都使用lock_guard来管理我的锁了，那**应该不会发生死锁了吧**。

实际上还是**可能发生死锁**。

比如我们的原子变量mutex是传入某个函数的**成员变量**：
```cpp

void swap(Classname obj1,Classname obj2)
{
std::lock_guard guard1(obj1.mtx);
std::lock_guard guard2(obj1.mtx);
}
```

此时，死锁又可能发生了，假如**我们同时在多线程环境中调用它两次，而且每次调用参数输入颠倒，就可能发生死锁。**

比如：
```cpp
void swap(obj1,obj2)//thread1
void swap(obj2,obj1)//thread2
```
这两个线程可能会在某一时刻死锁。

解决方法是：
我们可以使用**C++11**的`std::lock`或者**C++17**的`std::scoped_lock`

区别是**std::lock不会离开作用域自动解锁**，而**std::scoped_lock会自动解锁**
它们都会使用**死锁避免算法**来保证它们顺序交换也不会发生死锁。

以下是它们的代码

**std::lock**
```cpp
void multi_lock()
{
    std::unique_lock<std::mutex> lock1(m1, std::defer_lock);
    std::unique_lock<std::mutex> lock2(m2, std::defer_lock);
    // 批量加锁，算法级死锁避免
    std::lock(lock1, lock2); 
}
```
**std::scoped_lock**
```cpp
void multi_lock() 
{
    // 批量加锁，算法级死锁避免
    std::scoped_lock lock(m1, m2)
}
```