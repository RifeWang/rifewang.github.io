+++
draft = false
date = 2021-03-09T00:00:00+08:00
title = "XSS vs CSRF"
description = "XSS vs CSRF"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# XSS vs CSRF

在本节中，我们将解释 `XSS` 和 `CSRF` 之间的区别，并讨论 `CSRF token` 是否有助于防御 `XSS` 攻击。


## XSS 和 CSRF 之间有啥区别

跨站脚本攻击 `XSS` 允许攻击者在受害者用户的浏览器中执行任意 JavaScript 。

跨站请求伪造 `CSRF` 允许攻击者伪造受害用户执行他们不打算执行的操作。

`XSS` 漏洞的后果通常比 `CSRF` 漏洞更严重：
- `CSRF` 通常只适用于用户能够执行的操作的子集。通常，许多应用程序都实现 `CSRF` 防御，但是忽略了暴露的一两个操作。相反，成功的 `XSS` 攻击通常可以执行用户能够执行的任何操作，而不管该漏洞是在什么功能中产生的。
- `CSRF` 可以被描述为一个“单向”漏洞，因为尽管攻击者可以诱导受害者发出 HTTP 请求，但他们无法从该请求中检索响应。相反，`XSS` 是“双向”的，因为攻击者注入的脚本可以发出任意请求、读取响应并将数据传输到攻击者选择的外部域。


## CSRF token 能否防御 XSS 攻击

一些 `XSS` 攻击确实可以通过有效使用 `CSRF token` 来进行防御。假设有一个简单的反射型 `XSS` 漏洞，其可以被利用如下：
```
https://insecure-website.com/status?message=<script>/*+Bad+stuff+here...+*/</script>
```

现在，假设漏洞函数包含一个 `CSRF token` :
```
https://insecure-website.com/status?csrf-token=CIwNZNlR4XbisJF39I8yWnWX9wX4WFoz&message=<script>/*+Bad+stuff+here...+*/</script>
```

如果服务器正确地验证了 `CSRF token` ，并拒绝了没有有效令牌的请求，那么该令牌确实可以防止此 XSS 漏洞的利用。这里的关键点是“跨站脚本”的攻击中涉及到了跨站请求，因此通过防止攻击者伪造跨站请求，该应用程序可防止对 `XSS` 漏洞的轻度攻击。

这里有一些重要的注意事项：
- 如果反射型 `XSS` 漏洞存在于站点上任何其他不受 `CSRF token` 保护的函数内，则可以以常规方式利用该 `XSS` 漏洞。
- 如果站点上的任何地方都存在可利用的 `XSS` 漏洞，则可以利用该漏洞使受害用户执行操作，即使这些操作本身受到 `CSRF token` 的保护。在这种情况下，攻击者的脚本可以请求相关页面获取有效的 `CSRF token`，然后使用该令牌执行受保护的操作。
- `CSRF token` 不保护存储型 `XSS` 漏洞。如果受 `CSRF token` 保护的页面也是存储型 `XSS` 漏洞的输出点，则可以以通常的方式利用该 `XSS` 漏洞，并且当用户访问该页面时，将执行 `XSS` 有效负载。