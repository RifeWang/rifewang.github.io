# CORS 和 Access-Control-Allow-Origin 响应头


# CORS 和 Access-Control-Allow-Origin 响应头

在本节中，我们将解释有关 `CORS` 的 `Access-Control-Allow-Origin` 响应头，以及后者如何构成 `CORS` 实现的一部分。

`CORS` 通过使用一组 HTTP 头部提供了同源策略的可控制放宽，浏览器允许访问基于这些头部的跨域请求的响应。


## 什么是 Access-Control-Allow-Origin 响应头？

`Access-Control-Allow-Origin` 响应头标识了跨域请求允许的请求来源，浏览器会将 `Access-Control-Allow-Origin` 与请求网站 origin 进行比较，如果两者匹配则允许访问响应。


## 实现简单的 CORS

`CORS` 规范规定了 web 服务器和浏览器之间交换的头内容，其中 `Access-Control-Allow-Origin` 是最重要的。当网站发起跨域资源请求时，浏览器将会自动添加 `Origin` 头，随后服务器返回 `Access-Control-Allow-Origin` 响应头。

例如，origin 为 `normal-website.com` 的网站发起了如下跨域请求：
```
GET /data HTTP/1.1
Host: robust-website.com
Origin : https://normal-website.com
```

服务器响应：
```
HTTP/1.1 200 OK
...
Access-Control-Allow-Origin: https://normal-website.com
```

浏览器将会允许 `normal-website.com` 网站代码访问响应，因为 `Access-Control-Allow-Origin` 与 `Origin` 匹配。

`Access-Control-Allow-Origin` 允许多个域，或者 `null` ，或者通配符 `*` 。但是没有浏览器支持多个 origin ，且通配符的使用有限制。


## 带凭证的跨域资源请求

跨域资源请求的默认行为是传递请求时不会携带如 cookies 和 Authorization 头等凭证的。然而，对于带凭证的跨域请求，服务器通过设置 `Access-Control-Allow-Credentials: true` 响应头可以允许浏览器读取响应。例如，某个网站使用 JavaScript 去控制发起请求时一起发送 cookies ：
```
GET /data HTTP/1.1
Host: robust-website.com
...
Origin: https://normal-website.com
Cookie: JSESSIONID=<value>
```

得到的响应为：
```
HTTP/1.1 200 OK
...
Access-Control-Allow-Origin: https://normal-website.com
Access-Control-Allow-Credentials: true
```

那么浏览器将会允许发起请求的网站读取响应，因为 `Access-Control-Allow-Credentials` 设置为了 `true`。否则，浏览器将不允许访问响应。


## 使用通配符放宽 CORS

`Access-Control-Allow-Origin` 头支持使用通配符 `*` ，如
```
Access-Control-Allow-Origin: *
```

注意：通配符不能与其他值一起使用，如下方式是非法的：
```
Access-Control-Allow-Origin: https://*.normal-website.com
```

幸运的是，基于安全考虑，通配符的使用是有限制的，你不能同时使用通配符与带凭证的跨域传输。因此，以下形式的服务器响应是不允许的：
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

因为这是非常危险的，这等于向所有人公开目标网站上所有经过身份验证的内容。


## 预检

为了保护遗留资源不受 CORS 允许的扩展请求的影响，预检也是 CORS 规范中的一部分。在某些情况下，当跨域请求包括非标准的 HTTP method 或 header 时，在进行跨域请求之前，浏览器会先发起一次 method 为 `OPTIONS` 的请求，并且对服务端响应的 `Access-Control-*` 之类的头进行初步检查，对比 origin、method 和 header 等等，这就叫预检。

例如，对使用 `PUT` 方法和 `Special-Request-Header` 自定义请求头的预检请求为：
```
OPTIONS /data HTTP/1.1
Host: <some website>
...
Origin: https://normal-website.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Special-Request-Header
```

服务器可能响应：
```
HTTP/1.1 204 No Content
...
Access-Control-Allow-Origin: https://normal-website.com
Access-Control-Allow-Methods: PUT, POST, OPTIONS
Access-Control-Allow-Headers: Special-Request-Header
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 240
```

这个响应的含义:
- `Access-Control-Allow-Origin` 允许的请求域。
- `Access-Control-Allow-Methods` 允许的请求方法。
- `Access-Control-Allow-Headers` 允许的请求头。
- `Access-Control-Allow-Credentials` 允许带凭证的请求。
- `Access-Control-Max-Age` 设置预检响应的最大缓存时间，通过缓存减少预检请求增加的额外的 HTTP 请求往返的开销。


## CORS 能防止 CSRF 吗？

CORS 无法提供对跨站请求伪造（CSRF）攻击的防护，这是一个容易出现误解的地方。

CORS 是对同源策略的受控放宽，因此配置不当的 CORS 实际上可能会增加 CSRF 攻击的可能性或加剧其影响。
