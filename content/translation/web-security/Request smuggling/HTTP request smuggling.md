+++
draft = false
date = 2021-03-04T01:00:00+08:00
title = "web 安全之 HTTP request smuggling"
description = "web 安全之 HTTP request smuggling"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# HTTP request smuggling

在本节中，我们将解释什么是 HTTP 请求走私，并描述常见的请求走私漏洞是如何产生的。


## 什么是 HTTP 请求走私

HTTP 请求走私是一种干扰网站处理多个 HTTP 请求序列的技术。请求走私漏洞危害很大，它使攻击者可以绕过安全控制，未经授权访问敏感数据并直接危害其他应用程序用户。

![request smuggling](https://raw.githubusercontent.com/RifeWang/images/master/web-security/req-smuggling.png)


## HTTP 请求走私到底发生了什么

现在的应用架构中经常会使用诸如负载均衡、反向代理、网关等服务，这些服务在链路上起到了一个转发请求给后端服务器的作用，因为位置位于后端服务器的前面，所以本文把他们称为前端服务器。

当前端服务器（转发服务）将 HTTP 请求转发给后端服务器时，它通常会通过与后端服务器之间的同一个网络连接发送多个请求，因为这样做更加高效。协议非常简单：HTTP 请求被一个接一个地发送，接受请求的服务器则解析 HTTP 请求头以确定一个请求的结束位置和下一个请求的开始位置，如下图所示：
![http flow](https://raw.githubusercontent.com/RifeWang/images/master/web-security/request-smuggling-flow1.png)

在这种情况下，前端服务器（转发服务）与后端系统必须就请求的边界达成一致。否则，攻击者可能会发送一个模棱两可的请求，该请求被前端服务器（转发服务）与后端系统以不同的方式解析：

![http flow with smuggling](https://raw.githubusercontent.com/RifeWang/images/master/web-security/request-smuggling-flow2.png)

如上图所示，攻击者使上一个请求的一部分被后端服务器解析为下一个请求的开始，这时就会干扰应用程序处理该请求的方式。这就是请求走私攻击，其可能会造成毁灭性的后果。


## HTTP 请求走私漏洞是怎么产生的

绝大多数 HTTP 请求走私漏洞的出现是因为 HTTP 规范提供了两种不同的方法来指定请求的结束位置：`Content-Length` 头和 `Transfer-Encoding` 头。

`Content-Length` 头很简单，直接以字节为单位指定消息体的长度。例如：
```
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

`Transfer-Encoding` 头则可以声明消息体使用了 `chunked` 编码，就是消息体被拆分成了一个或多个分块传输，每个分块的开头是当前分块大小（以十六进制表示），后面紧跟着 `\r\n`，然后是分块内容，后面也是 `\r\n`。消息的终止分块也是同样的格式，只是其长度为零。例如：
```
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

b
q=smuggling
0
```

由于 HTTP 规范提供了两种不同的方法来指定 HTTP 消息的长度，因此单个消息中完全可以同时使用这两种方法，从而使它们相互冲突。HTTP 规范为了避免这种歧义，其声明如果 `Content-Length` 和 `Transfer-Encoding` 同时存在，则 `Content-Length` 应该被忽略。当只有一个服务运行时，这种歧义似乎可以避免，但是当多个服务被连接在一起时，这种歧义就无法避免了。在这种情况下，出现问题有两个原因：
- 某些服务器不支持请求中的 `Transfer-Encoding` 头。
- 某些服务器虽然支持 `Transfer-Encoding` 头，但是可以通过某种方式进行混淆，以诱导不处理此标头。

如果前端服务器（转发服务）和后端服务器处理 `Transfer-Encoding` 的行为不同，则它们可能在连续请求之间的边界上存在分歧，从而导致请求走私漏洞。


## 如何进行 HTTP 请求走私攻击

请求走私攻击需要在 HTTP 请求头中同时使用 `Content-Length` 和 `Transfer-Encoding`，以使前端服务器（转发服务）和后端服务器以不同的方式处理该请求。具体的执行方式取决于两台服务器的行为：
- `CL.TE`：前端服务器（转发服务）使用 `Content-Length` 头，而后端服务器使用 `Transfer-Encoding` 头。
- `TE.CL`：前端服务器（转发服务）使用 `Transfer-Encoding` 头，而后端服务器使用 `Content-Length` 头。
- `TE.TE`：前端服务器（转发服务）和后端服务器都使用 `Transfer-Encoding` 头，但是可以通过某种方式混淆标头来诱导其中一个服务器不对其进行处理。


### CL.TE 漏洞

前端服务器（转发服务）使用 `Content-Length` 头，而后端服务器使用 `Transfer-Encoding` 头。我们可以构造一个简单的 HTTP 请求走私攻击，如下所示：

<code>
    <span style="color:blue">
        POST / HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Length: 13<br>
        Transfer-Encoding: chunked<br>
        <br>
        0<br>
    </span>
    <br>
    <span style="color:orange">SMUGGLED</span>
</code>

前端服务器（转发服务）使用 `Content-Length` 确定这个请求体的长度是 13 个字节，直到 `SMUGGLED` 的结尾。然后请求被转发给了后端服务器。

后端服务器使用 `Transfer-Encoding` ，把请求体当成是分块的，然后处理第一个分块，刚好又是长度为零的终止分块，因此直接认为消息结束了，而后面的 `SMUGGLED` 将不予处理，并将其视为下一个请求的开始。


### TE.CL 漏洞

前端服务器（转发服务）使用 `Transfer-Encoding` 头，而后端服务器使用 `Content-Length` 头。我们可以构造一个简单的 HTTP 请求走私攻击，如下所示：

<code>
    <span style="color:blue">
        POST / HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Length: 3<br>
        Transfer-Encoding: chunked<br>
        <br>
        8<br>
    </span>
    <span style="color:orange">
        SMUGGLED<br>
        0<br>
        <br>
    </span>
</code>

注意：上面的 0 后面还有 `\r\n\r\n` 。

前端服务器（转发服务）使用 `Transfer-Encoding` 将消息体当作分块编码，第一个分块的长度是 8 个字节，内容是 `SMUGGLED`，第二个分块的长度是 0 ，也就是终止分块，所以这个请求到这里终止，然后被转发给了后端服务。

后端服务使用 `Content-Length` ，认为消息体只有 3 个字节，也就是 `8\r\n`，而剩下的部分将不会处理，并视为下一个请求的开始。


## TE.TE  混淆 TE 头

前端服务器（转发服务）和后端服务器都使用 `Transfer-Encoding` 头，但是可以通过某种方式混淆标头来诱导其中一个服务器不对其进行处理。

混淆 `Transfer-Encoding` 头的方式可能无穷无尽。例如：
```
Transfer-Encoding: xchunked

Transfer-Encoding : chunked

Transfer-Encoding: chunked
Transfer-Encoding: x

Transfer-Encoding:[tab]chunked

[space]Transfer-Encoding: chunked

X: X[\n]Transfer-Encoding: chunked

Transfer-Encoding
: chunked
```

这些技术中的每一种都与 HTTP 规范有细微的不同。实现协议规范的实际代码很少以绝对的精度遵守协议规范，并且不同的实现通常会容忍与协议规范的不同变化。要找到 `TE.TE` 漏洞，必须找到 `Transfer-Encoding` 标头的某种变体，以便前端服务器（转发服务）或后端服务器其中之一正常处理，而另外一个服务器则将其忽略。

根据可以混淆诱导不处理 `Transfer-Encoding` 的是前端服务器（转发服务）还是后端服务，而后的攻击方式则与 `CL.TE` 或 `TE.CL` 漏洞相同。


## 如何防御 HTTP 请求走私漏洞

当前端服务器（转发服务）通过同一个网络连接将多个请求转发给后端服务器，且前端服务器（转发服务）与后端服务器对请求边界存在不一致的判定时，就会出现 HTTP 请求走私漏洞。防御 HTTP 请求走私漏洞的一些通用方法如下：
- 禁用到后端服务器连接的重用，以便每个请求都通过单独的网络连接发送。
- 对后端服务器连接使用 HTTP/2 ，因为此协议可防止对请求之间的边界产生歧义。
- 前端服务器（转发服务）和后端服务器使用完全相同的 Web 软件，以便它们就请求之间的界限达成一致。

在某些情况下，可以通过使前端服务器（转发服务）规范歧义请求或使后端服务器拒绝歧义请求并关闭网络连接来避免漏洞。然而这种方法比上面的通用方法更容易出错。