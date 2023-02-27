+++
draft = false
date = 2019-09-25T00:56:24+08:00
title = "Go 垃圾回收"
description = "Go 垃圾回收"
slug = ""
authors = []
tags = ["Golang"]
categories = ["Golang"]
externalLink = ""
series = []
disableComments = true
+++


Garbage Collection（ GC ）也就是垃圾回收到底是什么？内存空间是有限的，诸如变量等需要分配内存才能存储数据，而当这个变量不再使用的时候就需要释放它占用的内存，这就是垃圾回收。

Go 的垃圾回收运行在后台的守护线程中，会自动追踪检查对象的使用情况，然后回收不再使用的空间，我们一般并不会也不需要直接接触到它。

## GC 模型

Go 使用的是 Mark-Sweep（标记-清除）方式，其具体的垃圾回收算法一直都在调整优化，本文并不打算去介绍这些算法，而是从一个整体的角度去描述 GC 的过程。


Collection 可以分为三个阶段：

- Mark Setup - STW
- Marking - Concurrent
- Mark Termination - STW


STW 是 Stop The World 的缩写，意思是 GC 的时候会暂停其它所有任务，正是如此才导致了延迟的存在。


### 1、Mark Setup - STW

垃圾回收开始，首先需要开启 Write Barrier（写屏障），为此所有应用程序 goroutine 必须暂停，这个过程通常很快，平均 10 - 30 微秒。


假设应用程序当前运行了四个 goroutine :

![](/images/go/gc1.png)

我们需要等待所有 goroutine 暂停，而暂停操作是需要出现一次函数调用才能完成，如果某个 goroutine 始终没有发生函数调用（比如一直在执行某个非常长的循环操作）而其它 goroutine 却完成了会怎样，就会如下图：

![](/images/go/gc2.png)

然而，必须所有的 goroutine 全部都暂停，垃圾回收才能继续进行，不然就会卡在这里一直等待，结果就是延迟越来越高。这个问题官方团队计划将在 1.14 版本通过优先策略进行优化。

一旦这一阶段完成，Write Barrier（写屏障）开启，就会进入下一阶段。


### 2、Marking - Concurrent

进行标记，Concurrent 表示这个过程是并发进行的，不会 STW ，GC 会先征用 25% 的 CPU 资源，如下图：

![](/images/go/gc3.png)

GC 占用了 P1 逻辑处理器，而其它 goroutine 正常的并发运行。


但是，有些时候 GC 的任务特别繁重，需要更多的资源，这个时候怎么办？开启 Mark Assit 协助工作，如下图中的 MA ：

![](/images/go/gc4.png)

标记完成，进行下一个阶段。


### 3、Mark Termination - STW


标记终止。关闭 Write Barrier（写屏障），执行各种清理任务，然后计算下一次 GC 的目标，这个阶段也是需要 STW 的，平均 60 - 90 微秒：

![](/images/go/gc5.png)


一旦 GC 完成，goroutine 继续执行：

![](/images/go/gc6.png)

### Sweeping - Concurrent

Sweeping（清除）需要等待 collection 完成之后，回收被标记为未使用的值的内存，这个过程发生在应用程序 goroutine 尝试给新值分配内存空间时，Sweeping 的延迟将会增加内存分配的成本。

---

## 延迟优化

虽然 Go 的 GC 很优秀，但正如前文所述，GC 的延迟还是会拖累应用程序的，那么我们在应用程序中可以进行怎么的优化呢？
答案是降低内存的压力即分配内存的频率，比如使用 slice 时，尽量避免因为容量不够了而导致分配更多的内存的频率。

如何调试我们的程序去发现需要优化的地方？

1、开启 gotrace 追踪各种指标：

```
GODEBUG=gctrace=1
```

通过指标数据可以看到各个过程及耗时情况，比如：

![](/images/go/gc7.jpeg)


2、使用 pprof

具体用法请自行参考其它资料。


---------

## 参考资料

- https://www.ardanlabs.com/blog/2018/12/garbage-collection-in-go-part1-semantics.html
- https://www.ardanlabs.com/blog/2019/05/garbage-collection-in-go-part2-gctraces.html
- https://www.ardanlabs.com/blog/2019/07/garbage-collection-in-go-part3-gcpacing.html
