# 深入理解 Node.js 事件循环架构【译】



*本文翻译自：
https://medium.com/preezma/node-js-event-loop-architecture-go-deeper-node-core-c96b4cec7aa4*


![](/images/uncate/node-event-loop1.jpeg)

关于 Node.js ，相信你已经了解过不少内容，诸如 Node.js 内核、事件循环、单线程、setTimeout 或 setImmediate 函数的执行机制等等。

当然最重要的，你应该知道 Node.js 使用的是非阻塞 IO 模型以及异步的编程风格。本文仍将深入核心进行相关内容的探讨。

## 01

事件循环到底是什么？Node.js 到底是单线程还是多线程？

关于这个问题，网络上充斥着各种不清晰甚至错误的答案。本文将会深入 Node.js 内核，阐述它是如何实现的以及它的工作机制。 Node.js 并不仅仅只是 " JavaScript on the Server " ，更重要的是，其中约 30% 的部分是 C++ 而不是 JS 。本文将会讲述这些 C++ 部分在 Node.js 中实际做了什么。


Node.js 是单线程？

答案：Node.js 既是单线程，但同时也不是。


一些相关名词：multitasking（多任务）、single-threaded（单线程）、multi-threaded（多线程），thread pool（线程池）、epoll loop（epoll 循环）、event loop（事件循环）。



让我们从头开始深入了解 Node.js 内核中发生了什么？

- 处理器可以一次处理一件事，也可以一次并行地处理多个任务（multitasking）。

对于单核处理器，其只能一次处理一个任务，应用程序在完成任务后调用 yield 去通知处理器开始处理下一个任务，就像 JavaScript 中的 generator 函数一样，否则没有 yield 则将返回当前任务。在过去，当应用程序无法调用 yield 时，其服务将处于无法访问的状态。

- 进程是一个 top level 执行容器，它有自己专用的内存系统。

这意味着在一个进程中无法直接获取另一个进程的内存中的数据，为了使两个进程进行通信，我们必须要另外做一些工作，称之为 inter-process communication（ IPC ，进程间通信），它依赖于 system sockets（系统套接字）。

Unix 系统中的工作基于 sockets 套接字。Socket 就是一个整数，返回一个 Socket() 系统调用，它被称为 socket descriptor（套接字描述符）或者 file descriptor（文件描述符）。

![](/images/uncate/node-event-loop2.jpeg)

Sockets 通过虚拟的接口（ read / write / pool / close 等）指向系统内核中的对象。

System sockets 系统套接字的工作方式类似于 TCP sockets ：将数据转换为 buffer 然后发送。由于我们在进行进程间通信时使用的是 JavaScript ，因此我们必须多次调用 JSON.stringify ，显然这是很低效的。

然而，我们拥有线程！

- 执行线程是可由调度器独立管理的最小程序指令序列。

线程在进程中运行，一个进程可以包含许多线程，并且由于这些线程处于同一个进程中，因此它们共享同一个内存。

这也就是说线程间通信不需要做任何额外的事情。如果我们在一个线程中托管一个全局变量，那么我们可以直接在另一个线程中访问它，因为它们都保持对同一个内存的引用，这种方式非常高效。

但是我们假设在一个线程中有一个函数，它写入一个 foo 变量，另一个线程则从中读取，这将会发生什么？

答案无从得知，因为我们无法确定读和写的先后顺序。这也正是多线程编程的难点所在。让我们看看 Node.js 如何处理这个问题。

Node.js 说：我只有一个线程。

实际上，Node.js 基于 V8 引擎，代码在主线程中执行，事件循环也运行在主线程中，这就是为什么我们说 Node.js 是单线程的。

但是，Node.js 不仅仅只是 V8，它有许多 APIs（C++），并且这些 API 都由 Event Loop 事件循环管理，通过 libuv（C++）实现。

C++ 在后台执行 JavaScript 代码并且拥有访问线程的权限。如果你执行从 Node.js 中调用的 JavaScript 同步方法，它将始终在主线程中运行。但是如果你执行一些异步的任务，它不会总是在主线程中执行：根据你使用的方法，事件循环可以将它路由到 APIs 中的某一个，并且它可以在另一个线程中执行。

看一个示例 CRYPTO ，它有许多 CPU 密集型方法，一些是同步的，一些是异步的。这里看一下 pbkdf2 方法。如果我们在 2 核处理器中执行其同步版本并进行 4 次调用，假设一次调用的执行时间是 2 ms ，则总耗时为 4 * 2 ms = 8 ms 。

但是如果在同一个 CPU（2核）中执行这个方法的异步版本，总耗时则为 2 * 2 ms = 4 ms ，因为处理器将使用默认 4 个线程（下文将会说明），将它托管到两个进程中并执行。

![](/images/uncate/node-event-loop3.jpeg)

这也就是：Node.js 并发地执行异步方法。

Node.js 使用一组预先分配的线程，称之为线程池，如果我们没有指定要打开的线程数，它默认就是使用 4 个线程。

我们可以通过 UV_THREADPOOL_SIZE 进行设置。

所以，Node.js 是多线程的吗？
当然，Node.js 使用了多线程。
然而，Node.js 到底是单线程还是多线程，这取决于 when ？


## 02

我们来看看 TCP 连接。

Thread per connection ：

![](/images/uncate/node-event-loop4.png)

创建一个 TCP server 最简单的方式就是创建一个 socket ，绑定这个 socket 到某个端口上然后 listen 监听。

![](/images/uncate/node-event-loop5.png)

在我们调用 listen 之前，该 socket 可用于建立连接或接受连接。当我们调用 listen 时，我们准备接受连接。

![](/images/uncate/node-event-loop6.png)

当连接到达并且我们需要写入它时，直到我们完成写入之前，我们都无法接受另一个连接，这就是我们将它推入另一个线程的原因。所以我们将 socket descriptor 和 function pointer 传递给线程。

现在，系统可以轻松处理几千个线程，但在这种情况下，我们必须为每个连接向线程发送大量数据，并且这样做并不能很好的扩展到两万到四万个并发连接。

但是，我们实际需要的仅仅只是 socket descriptor 套接字描述符，并记住我们要做的事情（也就是如何使用这些套接字）。所以有一种更好的方法：使用 Epoll（unix系统）或着 Kqueue（BSD系统，其实跟 Epoll 是同一个东西，不同系统名称不一样而已）。

Epoll 是 unix 系统相关底层知识。


Epoll 循环：

![](/images/uncate/node-event-loop7.png)

Epoll 能为我们带来什么，为什么要使用它。使用 Epoll 允许我们告诉 Kernel（系统内核）我们关注的事件，并且 Kernel 将会告诉我们这些事件何时发生。在上面的例子中，我们关注的是传入的 TCP 连接，因此，我们创建一个 Epoll 描述符并将其添加到 Epoll 循环中，并调用 wait 。每当有 TCP 连接传入时便会唤醒，然后将它添加到 Epoll 循环中并等待来自它的数据。这就是事件循环为我们做的事情。


举个例子：

当我们通过 http 请求向同一个 2 核处理器下载数据时，4 个，6 个，甚至 8 个请求需要的时间相同。这意味着什么？这意味着这里的限制与我们在线程池中的限制不同。

因为操作系统负责下载，我们只是要求它下载，然后问它：完成了吗？还没好吗？完成了吗？（监听 Epoll 中的 data 事件）。


## 03

APIs

哪些 API 对应于哪种方式呢？（线程，Epoll）

所有 fs.* 方法使用 uv thread pool，除非是同步方法。阻塞调用由线程完成，完成后将信号发送回事件循环。我们无法直接在 Epoll 中 wait ，只能 pipe 。Pipe 管道连接两端：一端是线程，当它完成时，往管道中写入数据，另一端在 Epoll 循环中等待，当它获取到数据时，Epoll 循环唤醒。因此 pipe 是由 Epoll 响应的。

一些主要的方法及其对应的响应方式：

EPOLL ：

- TCP/UDP servers and clients
- pipes
- dns.resolve


NGINX ：

- nginx signals ( sigterm )
- Child processes ( exec, spawn )
- TTY input ( console )


THREAD POOL ：

- fs.
- dns.lookup


事件循环负责发送和接受结果，如同中央调度器一般，将请求路由到 C++ API，然后将结果返回给 JavaScript 。


## 04

Event loop

事件循环到底是什么？它是一个无限的 while 循环，调用 Epoll wait 或者 pool ，当 Node.js 中我们关注的事情如 callback 回调、event 事件、fs 发生时，它将返回给 Node.js ，然后当 Epoll 不再有 wait 时退出。这就是 Node.js 中的异步工作方式，以及为什么我们称之为事件驱动。事件循环允许 Node.js 执行非阻塞 IO 操作。尽管 JavaScript 是单线程的，但只要有可能就会将操作丢给系统内核。

事件循环的一次迭代称之为 Tick，它有自己的 phases（阶段）。

![](/images/uncate/node-event-loop8.jpeg)

更多关于 event loop 的 phases、Timers、process.nextTick() 等请查阅官方文档。


## 05

Node.js v10.5.0 版本之后，新增了 worker_threads 工作线程模块，允许用户多线程并行执行 JavaScript 。

工作线程对于执行 CPU 密集型 JavaScript 操作非常有用，但对于 IO 密集型工作没有多大帮助，因为 Node.js 内置的异步 IO 操作比这些 workers 更高效。

