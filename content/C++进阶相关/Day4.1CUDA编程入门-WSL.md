---
title: Day4.1CUDA编程入门-WSL
date: 2026-01-04
tags:
  - Multi-Threading
  - CUDA
  - kernel
---

今天学习**WSL环境下的CUDA编程**

首先需要在WSL下载CUDA Tookit 需要与Nvdia的cuda驱动是**版本对齐**的，比如执行
```
nvidia-smi
```
发现我的**CUDA版本是12.8**，我下载了12.8的tookit
执行
```
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-8 //12.8
```
成功安装tookit

了解到**CUDA的语法基本是C/C++的拓展**。有一个重要的关键词 <mark style="background:rgba(240, 200, 0, 0.2)">__global__</mark>

在函数前注明它的将会编译为一个核函数，**核函数是CUDA编程的基础**
它会运行在GPU的线程上，相当于启动了一个**GPU版的thread**

以下是CUDA的一些知识

> [!note]
> ### 1. 线程 (Thread)：最小的工人
> 
> **线程**是 CUDA 中的最小执行单位。 每个线程执行一段相同的代码（即你写的 Kernel 函数），但处理不同的数据。例如：在处理图片时，一个线程只负责计算一个像素点的颜色。


---

> [!note]
> ### 2. 线程块 (Thread Block)：施工小组
> 
> **线程块**是一组线程的集合。
> 
> - **协作**：同一个块内的线程可以互相“交流”（通过共享内存 **Shared Memory**）并进行同步。
>     
> - **限制**：一个线程块内的线程数量有限制（目前主流显卡通常上限是 **1024** 个线程）。
>     
> - **独立性**：不同的线程块之间是**完全独立**的，它们不需要知道对方在做什么，这保证了 CUDA 可以在不同规模的 GPU 上运行。
>     


---

> [!note]
> ### 3. SM (Streaming Multiprocessor)：车间/流水线
> 
> **SM（流式多处理器）** 是 GPU 硬件层面的核心单元。
> 
> - **硬件 vs 软件**：线程块是**软件逻辑**上的概念，而 SM 是**物理硬件**上的概念。
>     
> - **分配关系**：当你启动一个 Kernel 时，GPU 的调度器会将你定义的**线程块**分配到不同的 **SM** 上去执行。
>     
> - **并发能力**：一块显卡拥有多少个 SM，决定了它能同时处理多少个任务。例如：
>     
>     - RTX 3060 约有 28 个 SM。
>         
>     - RTX 4090 约有 128 个 SM。
> 

那么这时候问题就来了，既然SM是有限的，所以GPU大量的核心**其实不是真正并发**的。

它牵扯到一个概念：**延迟隐藏”（Latency Hiding）**。

在 SM 硬件内部，基本的执行单位不是单个 Thread，也不是整个 Block，而是一个叫 **Warp（线程束）** 的东西。

- **1 Warp = 32 个线程**。
    
- SM 内部的硬件调度器每次只发布一条指令给这 32 个线程，让他们同步执行（这叫 **SIMT：单指令多线程**）。


GPU 之所以强大，是因为它进行了Wrap的**调度**：

1. **等待延迟**：当一个 Warp 执行指令时，如果需要从显存读取数据， Warp 就会阻塞，这可能需要消耗几百个时钟周期。
    
2. **快速切换**：此时，SM 的硬件调度器会**瞬间切换**到另一个已经准备好数据的 Warp 上去运行，切换时间几乎为零。
    
3. **延迟隐藏**：通过维持大量的线程，SM 可以保证在任何时刻，总有某些 Warp 是处于“可干活”状态的，从而把显存读取的漫长等待时间“藏”在其他 Warp 的计算时间里。


我定义了一个简单的Helloworld
```c
#include <stdio.h>  
#include <cuda_runtime.h>  
  
__global__ void helloCUDA() {  
    printf("Hello from block %d, thread %d\n", blockIdx.x, threadIdx.x);  
}  
  
void run_cuda_kernel() {  
    helloCUDA<<<20, 1024>>>();  
    cudaDeviceSynchronize();  
}
```
也就是说会有20个线程块，每一个线程块启动1024线程

使用性能分析工具 `nsys profile --stats=true --trace=cuda,nvtx ./project`发现只运行了300ms，而且有40%的时间是sync等待时间，CUDA真是相当快啊。