---
title: Day7.1C++多种线程读安全的单例模式
date: 2026-01-14
tags:
  - Multi-Threading
  - BestPractices
  - "#c-plus-plus"
---
今天学习c++的多种线程安全的单例模式
第一种是饿汉式线程单例模板类
它的主要特点是在程序开始时进行初始化，确保有对象使用它时它是初始化好的。
代码在这里
```cpp
template <typename T>
class Hungry_Singleton {
private:
    static T _instance;  // 程序启动时就创建
    Hungry_Singleton() = default;
public:
    static T& GetInstance() {
        return _instance;
    }
};

template <typename T>
T Hungry_Singleton<T>::_instance;  // 类外初始化，main之前执行//饿汉式，麻烦
```
但是比较麻烦的是在每个程序开始写的时候都要初始化一次，这在微服务场景是可能忘记引发错误的，
所以又引出了懒汉式单例模板，它的特点是使用双重检查锁来保证对象的初始化只有一次。
但是CPP的编译器优化原因，

new的时候可能会指令重排
导致第一次判断为非nullptr未初始化完成也会执行，导致UB行为，代码如下

```cpp
  
template <typename T>  
class Safe_Single  
{  
public:  
    std::shared_ptr<T> GetIn()  
    {  
        if(single!=nullptr)  
        {  
            return single;  
        }  
        _mutex.lock();  
        if(single!=nullptr)  
        {  
            _mutex.unlock();  
            return single;  
        }  
        single= std::make_shared<T>();  
        _mutex.unlock();  
  
        return single;  
    }  
    Safe_Single(const Safe_Single0<T>&)=delete;  
    Safe_Single<T> operator=(const Safe_Single0<T>&)=delete;  
private:  
    static std::mutex _mutex;  
   static std::shared_ptr<T> single;  
    Safe_Single()=default;  
      
};//存在指令重排的single空指针或UB行为

```
比如说在执行 ` single= std::make_shared<T>();  `时
new的底层分为三个阶段
第一个阶段为allocate 分配指定的空间。
第二个是construct调用构造函数。
第三个是赋值。
由于编译器优化的问题第二第三顺序是不确定的，所以可能导致UB。


另一个保证安全的方式是使用std::call_once来确保只会执行一次new。
代码如下
```cpp
  
template <typename T>  
  
class Safe_Callonce  
{  
private:  
    Safe_Callonce()=default;  
  
    inline static std::shared_ptr<T> _instance=nullptr;  
  
public:  
    Safe_Callonce(const Safe_Callonce<T>&)=delete;  
    Safe_Callonce<T>& operator=(const Safe_Callonce<T>&)=delete;  
   static  std::shared_ptr<T>& GetInstance()  
    {  
        static std::once_flag flag;  
  
        std::call_once(flag,[]()  
        {  
            _instance=std::make_shared<T>();  
        }  
            );  
        return _instance;  
    }  
};//安全但不高效
```
它的原理是std::call_once在底层使用了状态机和原子变量来保证new只有一次

在第一个线程进入call_once会先修改原子状态量，然后把其他线程阻塞挂起，开始处理call_once内部的函数，然后再让其他变量判断flag。

但是还有一个最简洁优雅高效的模式，是c++11 的静态局部变量单例模板类
代码如下

```cpp
  
template <typename T>  
class Safe_Static  
{  
private:  
    Safe_Static()=default;  
      
public:  
    Safe_Static(const Safe_Static<T>&)=delete;  
    Safe_Static<T>& operator=(const Safe_Static<T>&)=delete;  
  static T& GetInstance()  
    {  
        static T _instance;  
        return _instance;  
    }  
};//最简洁高效的模式

```
为什么说它是c++11的方式
因为在c++11之前写这样的代码同样是不安全的，以下是c++11添加的规则

在 C++11 标准文档中，有一项专门的规定（**Section 6.7 [stmt.dcl] p4**）：

> "If control enters the declaration concurrently while the variable is being initialized, the concurrent execution shall wait for completion of the initialization."
> 
> **翻译：** 如果多个线程同时进入一个静态变量的声明处，而该变量正在初始化，那么这些并发线程必须**等待**初始化完成。

这种方式如此简洁优雅，像是专门为线程安全的单例准备的。