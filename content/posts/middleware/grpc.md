+++
draft = false
date = 2019-08-23T00:25:31+08:00
title = "微服务互通的桥梁: gRPC 入门示例"
description = "微服务互通的桥梁: gRPC 入门示例"
slug = ""
authors = []
tags = ["Golang"]
categories = ["Golang"]
externalLink = ""
series = []
disableComments = true
+++


RPC 是什么？Remote Procedure Call ，远程过程调用，一种通信协议。你可以理解为，在某台机器上调用另外一台机器上的服务或方法。

应用服务对外可以提供 REST 接口以供进行服务的调用，那么对于分布式系统内部的微服务之间的相互调用呢？REST 的方式仍然可行，但是效率不高，因此 RPC 出现了。

gRPC 是谷歌开源的一套 RPC 实现机制，低延迟、高性能，其基于 HTTP/2 和 Protocol Buffers 。HTTP/2 在现行 HTTP/1.1 的基础上进行了大量优化，比如由文本传输变为二进制传输，同时具有多路复用、双向流等等特点，总之就是更牛了。Protocol Buffers 是一个序列化或反序列化数据的协议，说白了就是文本数据与二进制数据之间的相互转换。


文本将会带你入门 gRPC ，并且提供 Node.js 和 Go 两个版本的示例。

---

## Protocol Buffers

服务之间相互调用需要定义统一的数据格式（比如请求和响应），同时还要声明具体的服务及其方法，因此我们首先要做的就是定义一个 `.proto` 后缀的文件。

示例：

![](/images/middleware/grpc1.png)

1、`syntax` 声明使用的 protocol buffers 协议版本，现行的是第三版。
2、`package` 声明自定义的包名，这里的 package 可以理解为 go 中的包，或者 node.js 中的 module 。
3、`message` 定义数据格式，比如这里的 ReqBody 是请求的数据，响应结果则是 UserOrders ，名称都是自定义的，message 可以嵌套使用，message 内部需要定义具体的字段名称和数据类型，字段需要从 1 开始依次编号，但是枚举类型比较特别，枚举值从 0 开始编号。通过 repeated 声明某个字段可以重复，也就是这个数据是一个数组的形式。
4、`service` 定义服务名称，`rpc` 定义该服务下具体的方法，以及请求和响应的数据格式。

这个示例定义的是，我有一个服务叫 RPCService ，这个服务有一个方法叫 QueryUserOrders ，调用这个方法需要传递的请求数据的格式是 ReqBody ，响应结果的数据格式是 UserOrders 。

很简单是不是，`.proto` 协议文件清晰的定义了 RPC 服务、服务下的方法、请求和响应的数据格式，而 RPC 服务的客户端和服务端则将根据这个协议进行相互。

下面将会构建 RPC 服务端响应数据，以及 RPC 客户端发起请求。


## Node.js 版本

在 Node.js 中使用 gRPC 非常简单，我们需要依赖 `grpc` 和 `@grpc/proto-loader` 这两个官方包。

1、构建 gRPC 服务端：

![](/images/middleware/grpc2.jpeg)

如图所示，我们需要导入前面定义好的 .proto 文件，同时由于语言本身数据类型的不同，可以设置类型转换，比如将 .proto 中定义的枚举类型转换为 node.js 中的 string 类型。

gRPC 服务端需要按照 .proto 的约定，绑定服务以及实现具体的方法，同时由于其底层基于 HTTP/2 协议通信，因此还需要监听一个具体的端口并且启动这个 gRPC 服务。


2、构建 gRPC 客户端发起 RPC 调用：

```
# --proto_path 源路径， --go_out 输出路径，一定要指明 plugins=grpc
protoc --proto_path=grpc --go_out=plugins=grpc:grpc test.proto
```

需要注意的是，包名、服务名、方法名必须和 .proto 文件定义的保持一致。


## Go 版本

与 Node.js 不同的是 Go 是一个静态语言，需要先编译才能运行，因此使用 gRPC 有一点不同，我们先要去官网
https://github.com/protocolbuffers/protobuf/releases
下载并安装 protoc（ protocol buffers 编译器）。

1、执行 protoc 指令：

![](/images/middleware/grpc3.png)

编译 .proto 文件生成 .pb.go 代码包，在后续的使用中需要导入这个代码包。


2、构造 gRPC 服务端：

![](/images/middleware/grpc4.jpeg)

3、构建 gRPC 客户端发起 RPC 调用：

![](/images/middleware/grpc5.png)

`protoc` 编译 `.proto` 文件生成的 `.pb.go` 代码包里面包含了所有的服务、方法、数据结构等等，在我们的 go 代码中引用它们即可。


## 结语

不论是 gRPC 的客户端还是服务端并没有限制具体的语言，这意味着你完全可以使用 node.js 客户端去调用 go 服务端，或者其它任意语言的组合。

但是 gRPC 官方当前支持的语言是有限的，只有 Android、C#、C++、Dart、Go、Java、Node、PHP、Python、Ruby、Web（ js + envoy ）。

其次，gRPC 并不是万能的，比如大数据集（单条消息超过 1 MB ）就不适合用 gRPC ，即使你可以通过分块流式的方法来实现，但是复杂度会成倍的增加。



---

## 参考资料

- https://developers.google.com/protocol-buffers/docs/overview
- https://www.grpc.io/docs/guides
- https://github.com/grpc/grpc-node
- https://github.com/grpc/grpc-go
