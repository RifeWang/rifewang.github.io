+++
draft = false
date = 2021-02-28T00:00:00+08:00
title = "Same-origin policy (SOP)"
description = "Same-origin policy (SOP)"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# Same-origin policy (SOP) - 同源策略

在本节中，我们将解释什么是同源策略以及它是如何实现的。


## 什么是同源策略？

同源策略是一种旨在防止网站互相攻击的 web 浏览器的安全机制。

同源策略限制一个源上的脚本访问另一个源的数据。

Origin 源由三个部分组成：`schema`、`domain`、`port` ，所谓的同源就是要求这三个部分全部相同。 例如下面这个 URL：
```
http://normal-website.com/example/example.html
```
其 `schema` 是 http，`domain` 是 `normal-website.com`，`port` 是 80 。下表显示了如果上述 URL 中的内容尝试访问其它源将会是什么情况：

| 访问的 URL | 是否可以访问 |
| :--- | :--- |
| `http://normal-website.com/example/` | 是，同源	|
| `http://normal-website.com/example2/` | 是，同源 |
| `https://normal-website.com/example/` | 否: scheme 和 port 都不同 |
| `http://en.normal-website.com/example/` | 否: domain 不同	 |
| `http://www.normal-website.com/example/` | 否: domain 不同	 |
| `http://normal-website.com:8080/example/` | 否: port 不同* |

*IE 浏览器将会允许访问，因为 IE 浏览器在应用同源策略时不考虑端口号。

## 为什么同源策略是必要的？

当浏览器从一个源发送 HTTP 请求到另一个源时，与另一个源相关的任何 cookie （包括身份验证会话cookie）也将会作为请求的一部分一起发送。这意味着响应将在用户会话中返回，并包含此特定用户的相关数据。如果没有同源策略，如果你访问了一个恶意网站，它将能够读取你 GMail 中的电子邮件、Facebook 上的私人消息等。

## 同源策略是如何实施的？

同源策略通常控制 JavaScript 代码对跨域加载的内容的访问。通常允许页面资源的跨域加载。例如，同源策略允许通过 `<img>` 标签嵌入图像，通过 `<video>` 标签嵌入媒体、以及通过 `<script>` 标签嵌入 JavaScript 。但是，页面只能加载这些外部资源，页面上的任何 JavaScript 都无法读取这些资源的内容。

同源策略也有一些例外：
- 有些对象跨域可写入但不可读，例如 `location` 对象，或者来自 iframes 或新窗口的 `location.href` 属性。
- 有些对象跨域可读但不可写，例如 `window` 对象的 `length` 属性和 `closed` 属性。
- 在 `location` 对象上可以跨域调用 `replace` 函数。
- 你可以跨域调用某些函数。例如，你可以在一个新窗口上调用 `close`、`blur`、`focus` 函数。也可以在 iframes 和新窗口上 `postMessage` 函数以将消息从一个域发送到另一个域。

由于历史遗留，在处理 cookie 时，同源策略更为宽松，通常可以从站点的所有子域访问它们，即使每个子域并不满足同源的要求。你可以使用 `HttpOnly` 一定程度缓解这个风险。

使用 `document.domain` 可以放宽同源策略，这个特殊属性允许放宽特定域的同源策略，但前提是它是 FQDN（fully qualified domain name）的一部分。例如，你有一个域名 `marketing.example.com`，并且你想读取 `example.com` 域的内容。为此，两个域都需要设置 `document.domain` 为 `example.com`，那么同源策略将会允许这里两个域之间的访问，尽管它们并不同源。在过去，你可以将 `document.domain` 设置为顶级域名如 `com`，以允许同一个顶级域名上的任何域之间的访问，但是现代浏览器已经不允许这么做了。
