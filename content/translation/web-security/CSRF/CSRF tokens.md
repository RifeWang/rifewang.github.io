+++
draft = false
date = 2021-03-09T00:00:00+08:00
title = "CSRF tokens"
description = "CSRF tokens"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# CSRF tokens

在本节中，我们将解释什么是 CSRF token，它们是如何防御的 CSRF 攻击，以及如何生成和验证CSRF token 。


## 什么是 CSRF token

CSRF token 是一个唯一的、秘密的、不可预测的值，它由服务端应用程序生成，并以这种方式传输到客户端，使得它包含在客户端发出的后续 HTTP 请求中。当发出后续请求时，服务端应用程序将验证请求是否包含预期的 token ，并在 token 丢失或无效时拒绝该请求。

由于攻击者无法确定或预测用户的 CSRF token 的值，因此他们无法构造出一个应用程序验证所需全部参数的请求。所以 CSRF token 可以防止 CSRF 攻击。


## CSRF token 应该如何生成

CSRF token 应该包含显著的熵，并且具有很强的不可预测性，其通常与会话令牌具有相同的特性。

您应该使用加密强度伪随机数生成器（PRNG），该生成器附带创建时的时间戳以及静态密码。

如果您需要 PRNG 强度之外的进一步保证，可以通过将其输出与某些特定于用户的熵连接来生成单独的令牌，并对整个结构进行强哈希。这给试图分析令牌的攻击者带来了额外的障碍。


## 如何传输 CSRF token

CSRF token 应被视为机密，并在其整个生命周期中以安全的方式进行处理。一种通常有效的方法是将令牌传输到使用 POST 方法提交的 HTML 表单的隐藏字段中的客户端。提交表单时，令牌将作为请求参数包含：
```
<input type="hidden" name="csrf-token" value="CIwNZNlR4XbisJF39I8yWnWX9wX4WFoz" />
```

为了安全起见，包含 CSRF token 的字段应该尽早放置在 HTML 文档中，最好是在任何非隐藏的输入字段之前，以及在 HTML 中嵌入用户可控制数据的任何位置之前。这可以对抗攻击者使用精心编制的数据操纵 HTML 文档并捕获其部分内容的各种技术。

另一种方法是将令牌放入 URL query 字符串中，这种方法的安全性稍差，因为 query 字符串：
- 记录在客户端和服务器端的各个位置；
- 容易在 HTTP Referer 头中传输给第三方；
- 可以在用户的浏览器中显示在屏幕上。

某些应用程序在自定义请求头中传输 CSRF token 。这进一步防止了攻击者预测或捕获另一个用户的令牌，因为浏览器通常不允许跨域发送自定义头。然而，这种方法将应用程序限制为使用 XHR 发出受 CSRF 保护的请求（与 HTML 表单相反），并且在许多情况下可能被认为过于复杂。

CSRF token 不应在 cookie 中传输。


## 如何验证 CSRF token

当生成 CSRF token 时，它应该存储在服务器端的用户会话数据中。当接收到需要验证的后续请求时，服务器端应用程序应验证该请求是否包含与存储在用户会话中的值相匹配的令牌。无论请求的HTTP 方法或内容类型如何，都必须执行此验证。如果请求根本不包含任何令牌，则应以与存在无效令牌时相同的方式拒绝请求。