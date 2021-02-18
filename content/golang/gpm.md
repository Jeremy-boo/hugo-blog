---
title: "Golang调度"
date: 2020-08-27T11:02:21+08:00
lastmod: 2020-08-27T11:02:21+08:00
draft: false
keywords: ["调度"]
description: ""
tags: []
categories: ["golang"]
author: "Jeremy-boo"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
reward: true


# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

<!--more-->
# 1. Golang调度原理解析
> Go语言优势得益于其在并发编程方面有强大的能力，比线程更轻量级别的goroutine,支持协程间通信的channel等设计；关于Golang调度，离不开操作系统进程、线程这些概念，接下来就让我们一起来探讨下Go调度方面的原理吧！

![header](/images/go/go-gmp.png)

## 1.1 操作系统进程、线程以及调度概念
### 1.1.1 进程
> 进程是一个具有一定独立功能的程序在一个数据集上的一次动态执行的过程，是操作系统进行资源分配和调度的一个独立单位，是应用程序的载体

进程一般由程序、数据集合和进程控制块这三部分组成

- 进程控制块(PCB)：描述进程的基本信息和运行状态，进程的唯一标识
- 程序：用于描述进程要完成的功能，是控制进程执行的指令集
- 数据集合： 程序在执行时所需要的数据和工作区
### 1.1.2 线程
> 线程是操作系统进行资源分配的调度基本单位，一个进程中可以有多个线程，他们共享进程资源；线程能够并发的执行(并发是说一个单独内核上每个线程会轮询占用一段cpu时间),而不是并行执行(在不同内核上同时执行)。线程同时会维持它自己的状态，并且能够在本地安全、独立地执行他自己的指令

线程可以分为:用户级线程和内核级线程

线程状态可分为：等待(waiting),可运行(runnable)以及执行中(Executing)

线程按照工作类型可分为：cpu密集型与io密集型两类

### 1.1.3 调度
进程调度算法：
- 先来先服务(FCFS):按照到达的先后顺序调度，也就是等待时间越久的越优先得到服务。
- 短作业优先(SJF):每次调度时选择当前已到达且运行时间最短的进程。
- 最短剩余时间优先(SRTN):短作业的一种，按照剩余时间最短的顺序进行调度。
- 时间片轮转:轮流让就绪队列中的队头的进程依次执行一个时间片
- 优先级调度:优先处理优先级最高的进程
- 多级反馈队列:设置多级就绪队列，各级队列优先级从高到低，时间片从小到大

上下文切换：
在内核上切换线程的物理行为叫做上下文切换(context switch)。上下文切换发生在这样的情况，调度器从内核换下正在执行的线程，替换上可执行的线程。线程是从运行队列中取出，并设置成执行中(Executing)的状态。从内核上下来的线程会置成可运行状态,或者是等待状态。
## 2. Go调度原理
> Go语言的调度器能有今天的优异性能，是工程师不断优化和改进取得的结果；从单线程调度器->多线程调度器->任务窃取调度器->如今的抢占式调度器

抢占式调度器：
- 基于协作的抢占式调度器 - 1.2 ~ 1.13

  - 通过编译器在函数调用时插入抢占检查指令，在函数调用时检查当前 Goroutine 是否发起了抢占请求，实现基于协作的抢占式调度；
  - Goroutine 可能会因为垃圾回收和循环长时间占用资源导致程序暂停；
- 基于信号的抢占式调度器 - 1.14 ~ 至今
  
   - 实现基于信号的真抢占式调度；
   - 垃圾回收在扫描栈时会触发抢占调度；
   - 抢占的时间点不够多，还不能覆盖全部的边缘情况；
### 2.1 调度模型(GPM)介绍
Go调度器内部有4个重要的结构，即M,P,G,Sched
- G: 代表一个goroutine,拥有自己的栈,instruction pointer和其他信息（正在等待的channel等等）,用于调度。
- P:全称processor，代表cpu处理器，用来执行goroutine; 每个P都有一个自己的runnableG队列(大小为256)，可以从里面拿出一个G执行，如果本地队列没有，可以从全局队列中获取，G通过P依附在M上执行；p的数量可以通过GOMAXPROCS()来设置
- M:指go中的工作线程，是真正执行代码的单元，goroutine就是运行在M上的

调度模型图如下：
![go-gpm](/images/go/go-gpm1.jpg)
### 2.2 结构模型
> 定义在 /src/runtime/runtime2.go中 

G结构体：

    type g struct {
      stack       stack   // 栈内存范围
      stackguard0 uintptr // 调度器抢占式调度
      preempt       bool  // 抢占信号
      preemptStop   bool  // 抢占时将状态修改成_Gpreempted 
      preemptShrink bool  // 在同步安全点收缩栈
      m              *m   // 当前Goroutine占用的线程
      sched          gobuf  // 调度相关信息
      atomicstatus   uint32 // gouroutine的状态
      goid           int64  // gourotine id
      ...
    }

sched字段的runtime2.gobuf结构内容：

    type gobuf struct {
      sp   uintptr   //  栈指针
      pc   uintptr   //  程序计数器
      g    guintptr  //  持有runtime.gobuf的goroutine
      ret  sys.Uintreg // 系统调用返回值
      ...
    }

M结构体：
> Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 GOMAXPROCS 个活跃线程能够正常运行。

    type m struct {
      g0      *g    // 持有调度栈的goroutine
      curg    *g    // 当前运行的goroutine
      p             puintptr // 正在运行代码的处理器
      nextp         puintptr // 暂停的处理器
      oldp          puintptr // 执行系统调用之前使用线程的处理器 
      ...
    }

g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。

P结构体：
> 调取器中的P是线程(M)和goroutine的中间层，提供线程需要的上下文环境，负责调度线程上的等待队列，通过P的调度，每一个内核线程都能执行多个goroutine,它能在goroutine进行一些I/O操作时及时切换，调高线程利用率

    type p struct{
      m           muintptr     // 回链关联的m    

      // 可运行的goroutine队列
      runqhead    uint32
      runqtail    uint32
      runq        [256]guintptr

      runnext     guintptr      // 线程下一个需要执行的goroutine
      ...
    }
### 2.3 调度器的实现
