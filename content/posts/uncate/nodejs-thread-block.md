+++
draft = false
date = 2019-09-24T17:19:31+08:00
title = "CPU 密集型任务会阻塞 Node.js 吗【译】"
description = "CPU 密集型任务会阻塞 Node.js 吗【译】"
slug = ""
authors = []
tags = ["Node.js"]
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

*本文翻译自：
https://betterprogramming.pub/is-node-js-really-single-threaded-7ea59bcc8d64*


CPU密集型任务会阻塞 Node.js 吗？


让我们使用加密任务做个简单测试：

![](/images/uncate/nodejs-block1.png)

如图所示，连续执行四次加密任务，打印耗时，结果会发生什么？


结果输出：

```
Hash:  1232
Hash:  1237
Hash:  1268
Hash:  1297
```

这四次加密任务计时的起始时间都是相同的，然后最终的结束时间却几乎一致，这个结果说明了什么？说明它们是并发执行的。

![](/images/uncate/nodejs-block2.png)


如果不是并发执行，那么结果就会如下图所示：

![](/images/uncate/nodejs-block3.jpeg)


那么为什么这里没有发生阻塞？

![](/images/uncate/nodejs-block4.jpeg)

Node.js 的执行过程如上图所示，我们要注意的是 libuv 默认使用了四个线程！上述示例中的四个加密任务分别推送到了四个不同的线程中去并发执行，所以才没有发生阻塞。


那么问题来了？如果连续执行五个加密任务呢？

![](/images/uncate/nodejs-block5.png)

输出结果：

```
Hash:  1432
Hash:  1437
Hash:  1468
Hash:  1497
Hash:  2104
```

可以看到前四个任务仍然是并发执行的，但是第五个任务发生了阻塞。

![](/images/uncate/nodejs-block6.png)

为什么？因此 libuv 的四个线程都在忙碌，第五个任务只有等待线程的任务执行完毕才能推送到线程中去执行。


过程如下图所示：

1、四个线程都在忙碌，其它任务必须等待：

![](/images/uncate/nodejs-block7.jpeg)

2、某个线程任务完成，继续执行其它任务：

![](/images/uncate/nodejs-block8.jpeg)

libuv 线程池中的线程数量是否可以设置？
通过环境变量 UV_THREADPOOL_SIZE 即可设置。

比如：

![](/images/uncate/nodejs-block9.png)

我把线程数设置为 5 ，执行的结果就会是下图所示：

![](/images/uncate/nodejs-block10.png)

请注意测试环境的 CPU 核心数是四个，需要说明的有两点：第一，五个任务被推送到了五个线程中去并发执行，这一点上文已经说明；第二，每个任务的耗时有了明显的增加，为什么？因为我们只有四核，但是却有五个线程，操作系统需要进行平衡调度、通过上下文切换以保证每个线程分配到相同的时间去执行任务。