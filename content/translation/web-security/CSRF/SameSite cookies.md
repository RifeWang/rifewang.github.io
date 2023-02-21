+++
draft = false
date = 2021-03-09T00:00:00+08:00
title = "SameSite cookies"
description = "SameSite cookies"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# SameSite cookies

某些网站使用 SameSite cookies 防御 `CSRF` 攻击。

这个 `SameSite` 属性可用于控制是否以及如何在跨站请求中提交 cookie 。通过设置会话 cookie 的属性，应用程序可以防止浏览器默认自动向请求添加 cookie 的行为，而不管cookie 来自何处。

这个 `SameSite` 属性在服务器的 `Set-Cookie` 响应头中设置，该属性可以设为 `Strict` 严格或者 `Lax` 松懈。例如：
```
SetCookie: SessionId=sYMnfCUrAlmqVVZn9dqevxyFpKZt30NN; SameSite=Strict;

SetCookie: SessionId=sYMnfCUrAlmqVVZn9dqevxyFpKZt30NN; SameSite=Lax;
```

如果 `SameSite` 属性设置为 `Strict` ，则浏览器将不会在来自其他站点的任何请求中包含cookie。这是最具防御性的选择，但它可能会损害用户体验，因为如果登录的用户通过第三方链接访问某个站点，那么他们将不会登录，并且需要重新登录，然后才能以正常方式与站点交互。

如果 `SameSite` 属性设置为 `Lax` ，则浏览器将在来自另一个站点的请求中包含cookie，但前提是满足以下两个条件：
- 请求使用 GET 方法。使用其他方法（如 POST ）的请求将不会包括 cookie 。
- 请求是由用户的顶级导航（如单击链接）产生的。其他请求（如由脚本启动的请求）将不会包括 cookie 。

使用 `SameSite` 的 `Lax` 模式确实对 `CSRF` 攻击提供了部分防御，因为 `CSRF` 攻击的目标用户操作通常使用 POST 方法实现。这里有两个重要的注意事项：
- 有些应用程序确实使用 GET 请求实现敏感操作。
- 许多应用程序和框架能够容忍不同的 HTTP 方法。在这种情况下，即使应用程序本身设计使用的是 POST 方法，但它实际上也会接受被切换为使用 GET 方法的请求。

出于上述原因，不建议仅依赖 `SameSite` Cookie 来抵御 `CSRF` 攻击。当其与 `CSRF token` 结合使用时，`SameSite` cookies 可以提供额外的防御层，并减轻基于令牌的防御中的任何缺陷。