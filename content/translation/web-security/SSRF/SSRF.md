+++
draft = false
date = 2021-03-01T01:00:00+08:00
title = "web 安全之 Server-side request forgery"
description = "web 安全之 Server-side request forgery"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# Server-side request forgery (SSRF)

在本节中，我们将解释 server-side request forgery（服务端请求伪造）是什么，并描述一些常见的示例，以及解释如何发现和利用各种 `SSRF` 漏洞。


## SSRF 是什么

`SSRF` 服务端请求伪造是一个 web 漏洞，它允许攻击者诱导服务端程序向攻击者选择的任何地址发起 HTTP 请求。

在典型的 `SSRF` 示例中，攻击者可能会使服务端建立一个到服务端自身、或组织基础架构中的其它基于 web 的服务、或外部第三方系统的连接。

![SSRF](https://raw.githubusercontent.com/RifeWang/images/master/web-security/ssrf.png)


## SSRF 攻击的影响

成功的 `SSRF` 攻击通常会导致未经授权的操作或对组织内部数据的访问，无论是在易受攻击的应用程序本身，还是应用程序可以通信的其它后端系统。在某些情况下，`SSRF` 漏洞可能允许攻击者执行任意的命令。

利用 `SSRF` 漏洞可能可以操作服务端应用程序使其向与之连接的外部第三方系统发起恶意请求，这将导致潜在的法律责任和声誉受损。


## 常见的 SSRF 攻击

`SSRF` 攻击通常利用服务端应用程序的信任关系发起攻击并执行未经授权的操作。这种信任关系可能包括：对服务端自身的信任，或同组织内其它后端系统的信任。


### SSRF 攻击服务端自身

在针对服务端本身的 `SSRF` 攻击中，攻击者诱导应用程序向其自身发出 HTTP 请求，这通常需要提供一个主机名是 `127.0.0.1` 或者 `localhost` 的 URL 。

例如，假设某个购物应用程序，其允许用户查看某个商品在特定商店中是否有库存。为了提供库存信息，应用程序需要通过 REST API 查询其他后端服务，而其他后端服务的 URL 地址直接包含在前端 HTTP 请求中。因此，当用户查看商品的库存状态时，浏览器可能发出如下请求：
```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```

这将导致服务端向指定的 URL 发出请求，检索库存状态，然后将结果返回给用户。

在这种情况下，攻击者可以修改请求以指定服务器本地的 URL ，例如：
```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```

此时，服务端将会访问本地 /admin URL 并将其内容返回给用户。

当然，攻击者可以直接访问 /admin URL ，但是这通常没用，因为管理功能基本都需要进行适当的身份验证，而如果对 /admin URL 的请求来自机器本地，则正常情况下的访问控制可能会被绕过。该服务端应用程序可能会授予对管理功能的完全访问权限，因为请求似乎来自受信任的位置。

为什么应用程序会以这种方式运行，并且隐式信任来自本地的请求？这可能有多种原因：
- 访问控制检查可能是另外的一个微服务。当服务器连接自身时，将会绕过访问控制检查。
- 出于灾难恢复的目的，应用程序可能允许来自本地机器的任何用户在不登录的情况下进行管理访问。这为管理员在丢失凭证时恢复系统提供了一种方法。这里的假设是只有完全可信的用户才能直接来自服务器本地。
- 管理接口可能与主应用是不同的端口号，因为用户可能无法直接访问。

在这种信任关系中，来自本地机器的请求的处理方式与普通请求不同，这常常使 `SSRF` 成为一个严重的漏洞。


### 针对其他后端系统的 SSRF 攻击

`SSRF` 利用的另外一种信任关系是应用服务端与用户无法直接访问的内部后端系统之间进行的交互，这些后端系统通常具有不可路由的专用 IP 地址，由于受到网络拓扑结构的保护，它们的安全性往往较弱。在许多情况下，内部后端系统包含一些敏感功能，任何能够与系统交互的人都可以在不进行身份验证的情况下访问这些功能。

在前面的示例中，假设后端系统有一个管理接口 `https://192.168.0.68/admin` 。此时，攻击者可以通过提交以下请求利用 SSRF 漏洞访问管理接口：
```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```


## 规避常见的 SSRF 防御

通常应用程序包含 SSRF 行为以及防止恶意攻击的防御措施，然而这些防御措施是可以被规避的。


### 基于黑名单过滤的 SSRF

某些应用程序禁止例如 `127.0.0.1`、`localhost` 等主机名、或 `/admin` 等敏感 URL 。这种情况下，可以使用各种技巧绕过过滤：
- 使用 `127.0.0.1` 的替代 IP 地址表示，例如 `2130706433`，`017700000001`，`127.1` 。
- 注册自己的域名，并解析为 `127.0.0.1` ，你可以直接使用 `spoofed.burpcollaborator.net` 。
- 使用 URL 编码或大小写变化来混淆被阻止的字符串。


### 基于白名单过滤的 SSRF

有些应用程序只允许输入匹配、或包含白名单中的值，或以白名单中的值开头。在这种情况下，有时可以利用 URL 解析的不一致来绕过过滤器。

URL 规范包含有许多在实现 URL 的解析和验证时容易被忽略的特性：
- 你可以在主机名之前使用 `@` 符号嵌入凭证。例如 `https://expected-host@evil-host` 。
- 你可以使用 `#` 符号表示一个 URL 片段。例如 `https://evil-host#expected-host` 。
- 你可以利用 DNS 命令层次结构将所需的输入放入你控制的标准 DNS 名称中。例如 `https://expected-host.evil-host` 。
- 你可以使用 URL 编码字符来迷惑 URL 解析代码。如果处理 URL 编码的过滤器的实现不同与执行后端 HTTP 请求的代码，这一点尤其有用。
- 你可以把这些技巧结合起来使用。


### 通过开放重定向绕过 SSRF 过滤器

有时利用开放重定向漏洞可以绕过任何基于过滤器的防御。

在前面的示例中，假设用户提交的 URL 经过严格验证，以防止恶意利用 SSRF 的行为，但是，允许使用 URL 的应用程序包含一个开放重定向漏洞。如果用于发起后端 HTTP 请求的 API 支持重定向，那么你可以构造一个满足过滤器的要求的 URL ，并将请求重定向到所需的后端目标。

例如，假设应用程序包含一个开放重定向漏洞，例如下面 URL 的形式：
```
/product/nextProduct?currentProductId=6&path=http://evil-user.net
```

重定向到：
```
http://evil-user.net
```

你可以利用开放重定向漏洞绕过 URL 过滤器，并利用 SSRF 漏洞进行攻击，如：
```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

这个 SSRF 攻击之所有有效，是因为首先 stockAPI URL 在应用程序允许的域上，然后应用程序向提供的 URL 发起请求，触发了重定向，最终向重定向的内部 URL 发起了请求。



## Blind SSRF - 不可见 SSRF 漏洞

所谓 Blind SSRF（不可见 SSRF）漏洞是指，可以诱导应用程序向提供的 URL 发起后端 HTTP 请求，但是请求的响应并没有在应用程序的前端响应中返回。

不可见 SSRF 漏洞通常较难利用，但有时会导致服务器或其他后端组件上的远程代码执行。



## 寻找 SSRF 漏洞的隐藏攻击面

许多 SSRF 漏洞之所以相对容易发现，是因为应用程序的正常通信中就包含了完整的 URL 请求参数。而其它情况就比较难搞了。


### 请求中的部分 URL

有时应用程序只将主机名或 URL 路径的一部分放入请求参数中，然后，提交的值被合并到服务端请求的完整 URL 中。如果该值很容易被识别为主机名或 URL 路径，那么潜在的攻击面可能很明显。但是，因为你不能控制最终请求的 URL，所以 SSRF 的可利用性会受到限制。


### 数据格式内的 URL

有些应用程序以某种数据格式传输数据，URL 则包含在指定数据格式中。这里的数据格式的一个明显的例子就是 XML ，当应用程序接受 XML 格式的数据并对其进行解析时，可能会受到 `XXE` 注入，进而通过 `XXE` 完成 `SSRF` 攻击。有关 `XXE` 注入漏洞会有专门的章节讲解。


### 通过 Referer 头的 SSRF

一些应用程序使用服务端分析软件来跟踪访问者，这种软件经常在请求中记录 `Referer` 头，因为这对于跟踪传入链接特别有用。通常，分析软件实际上会访问 `Referer` 头中出现的任何第三方 URL 。这通常用于分析引用站点的内容，包括传入链接中使用的锚文本。因此，`Referer` 头通常是 `SSRF` 漏洞的有效攻击面。有关涉及 `Referer` 头的漏洞示例请参阅 `Blind SSRF` 。