---
title: Day3.1C++原子变量学习
date: 2026-01-03
tags:
  - Multi-Threading
  - BestPractices
  - "#c-plus-plus"
  - mutex
  - Multi-Safe
  - lock_gurad
---
今天学习C++多线程开发的互斥量相关。

我写了一个看似线程安全的stack
```cpp
template<typename T>  
class weak_stack  
{  
public:  
    weak_stack()  
        : _stack()  
    {  
    }  
  
    void Pop()  
    {  
        _mutex.lock();  
        _stack.pop();  
        std::cout<<"栈大小:"<<_stack.size()<<std::endl;  
       _mutex.unlock();  
    }  
  
    void Push(const T param)  
    {  
       _mutex.lock();  
        _stack.push(param);  
        std::cout<<"栈大小:"<<_stack.size()<<std::endl;  
      _mutex.unlock();  
  
    }  
  
    bool Empty() const  
    {  
        return _stack.empty();  
    }  
  
private:  
    mutable std::mutex _mutex;  
    std::stack<T> _stack;  
};
```
**理论上来说这段代码长时间在多线程环境必然引发Undefined**
但是我开两个线程跑了半天居然没有报错？
```cpp
  std::jthread j1([&w1]()  
  {  
      Timer timer;  
      for(int i=0;i<=1000;i++)  
      {  
          w1.Push(i);  
          if (!w1.Empty())  
              w1.Pop();  
          std::this_thread::sleep_for(std::chrono::microseconds(10));  
      }  
  
  });  
std::jthread j2([&w1]()  
  {  
      Timer timer;  
      for(int i=0;i<=100;i++)  
      {  
          w1.Push(i);  
          if (!w1.Empty())  
              w1.Pop();  
          std::this_thread::sleep_for(std::chrono::microseconds(10));  
      }  
  });
```
好像是因为都是一个push一个pop导致崩溃的概率变小了。
于是我把下面jthread的push去掉了

**立刻引发了崩溃**
![[栈未定义.png]]
这是因为虽然**pop和push都加解锁**了。
但是外部判断empty是读行为，**读取行为在多线程可能是不安全的**，外部读数据，这个**数据的真实性并没有保证**，例如：
==假设虽然某一刻它为空，但是一个线程也判断为空，此时另一个线程也判断为空，然后此时栈还只剩一个元素！这时候就会出栈一个元素为空的栈，引发崩溃。==

我想了一个处理办法
```cpp
void Pop()  
{  
    _mutex.lock();  
    if (_stack.empty()) {  
        _mutex.unlock();  
        return;  
    }  
    _stack.pop();  
    std::cout<<"栈大小:"<<_stack.size()<<std::endl;  
   _mutex.unlock();  
}
```
在内部判断为空后直接return，但这样其实**外部判断就没有意义**了
所以也可以**抛出一个异常让外部处理**。

看这段代码的mutex实在是太臃肿而且容易错误，实际上**我刚刚就忘记在return前解锁导致死锁**，
所以最好还是使用自动加解锁的lock_gurad，本来我还认为在一个循环中频繁释放创建lock_gurad会有性能问题，但是使用**基于生命周期的计时器**发现，时间差距还<u>没有误差大</u>
```cpp
#include <chrono>  
  
class Timer  
{  
public:  
    Timer()  
    {  
        // 1. 在构造时记录起点  
        m_StartTimepoint = std::chrono::high_resolution_clock::now();  
    }  
  
    ~Timer()  
    {  
        // 2. 在析构时记录终点  
        auto endTimepoint = std::chrono::high_resolution_clock::now();  
  
        // 3. 计算差值  
        auto start = std::chrono::time_point_cast<std::chrono::microseconds>(m_StartTimepoint).time_since_epoch().  
                count();  
        auto end = std::chrono::time_point_cast<std::chrono::microseconds>(endTimepoint).time_since_epoch().count();  
  
        auto duration = end - start;  
        double ms = duration * 0.001; // 转换为毫秒方便阅读  
  
        std::cout << "耗时: " << duration << "us (" << ms << "ms)\n";  
    }  
  
private:  
    std::mutex _mutex;  
    std::chrono::time_point<std::chrono::high_resolution_clock> m_StartTimepoint;  
};
```
所以最佳实践还是使用自动加解锁的对象，就像std::unique_ptr。
**你无法永远保证你在一个大型项目记得你的new和delete是否是一对一的**