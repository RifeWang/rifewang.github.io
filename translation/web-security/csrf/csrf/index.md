# web 安全之 CSRF


# Cross-site request forgery (CSRF)

在本节中，我们将解释什么是跨站请求伪造，并描述一些常见的 `CSRF` 漏洞示例，同时说明如何防御 `CSRF` 攻击。


## 什么是 CSRF

跨站请求伪造（`CSRF`）是一种 web 安全漏洞，它允许攻击者诱使用户执行他们不想执行的操作。攻击者进行 `CSRF` 能够部分规避同源策略。

![](https://raw.githubusercontent.com/RifeWang/images/master/web-security/csrf.png)


## CSRF 攻击能造成什么影响

在成功的 `CSRF` 攻击中，攻击者会使受害用户无意中执行某个操作。例如，这可能是更改他们帐户上的电子邮件地址、更改密码或进行资金转账。根据操作的性质，攻击者可能能够完全控制用户的帐户。如果受害用户在应用程序中具有特权角色，则攻击者可能能够完全控制应用程序的所有数据和功能。


## CSRF 是如何工作的

要使 `CSRF` 攻击成为可能，必须具备三个关键条件：
- 相关的动作。攻击者有理由诱使应用程序中发生某种动作。这可能是特权操作（例如修改其他用户的权限），也可能是针对用户特定数据的任何操作（例如更改用户自己的密码）。
- 基于 Cookie 的会话处理。执行该操作涉及发出一个或多个 HTTP 请求，应用程序仅依赖会话cookie 来标识发出请求的用户。没有其他机制用于跟踪会话或验证用户请求。
- 没有不可预测的请求参数。执行该操作的请求不包含攻击者无法确定或猜测其值的任何参数。例如，当导致用户更改密码时，如果攻击者需要知道现有密码的值，则该功能不会受到攻击。

假设应用程序包含一个允许用户更改其邮箱地址的功能。当用户执行此操作时，会发出如下 HTTP 请求：
```
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Cookie: session=yvthwsztyeQkAPzeQ5gHgTvlyxHfsAfE

email=wiener@normal-user.com
```

这个例子符合 `CSRF` 要求的条件：
- 更改用户帐户上的邮箱地址的操作会引起攻击者的兴趣。执行此操作后，攻击者通常能够触发密码重置并完全控制用户的帐户。
- 应用程序使用会话 cookie 来标识发出请求的用户。没有其他标记或机制来跟踪用户会话。
- 攻击者可以轻松确定执行操作所需的请求参数的值。

具备这些条件后，攻击者可以构建包含以下 HTML 的网页：
```
<html>
  <body>
    <form action="https://vulnerable-website.com/email/change" method="POST">
      <input type="hidden" name="email" value="pwned@evil-user.net" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

如果受害用户访问了攻击者的网页，将发生以下情况：
- 攻击者的页面将触发对易受攻击的网站的 HTTP 请求。
- 如果用户登录到易受攻击的网站，其浏览器将自动在请求中包含其会话 cookie（假设 SameSite cookies 未被使用）。
- 易受攻击的网站将以正常方式处理请求，将其视为受害者用户发出的请求，并更改其电子邮件地址。

注意：虽然 `CSRF` 通常是根据基于 cookie 的会话处理来描述的，但它也出现在应用程序自动向请求添加一些用户凭据的上下文中，例如 HTTP Basic authentication 基本验证和 certificate-based authentication 基于证书的身份验证。


## 如何构造 CSRF 攻击

手动创建 `CSRF` 攻击所需的 HTML 可能很麻烦，尤其是在所需请求包含大量参数的情况下，或者在请求中存在其他异常情况时。构造 `CSRF` 攻击的最简单方法是使用 `Burp Suite Professional`（付费软件） 中的 `CSRF PoC generator`。


## 如何传递 CSRF

跨站请求伪造攻击的传递机制与反射型 XSS 的传递机制基本相同。通常，攻击者会将恶意 HTML 放到他们控制的网站上，然后诱使受害者访问该网站。这可以通过电子邮件或社交媒体消息向用户提供指向网站的链接来实现。或者，如果攻击被放置在一个流行的网站（例如，在用户评论中），则只需等待用户上钩即可。

请注意，一些简单的 `CSRF` 攻击使用 GET 方法，并且可以通过易受攻击网站上的单个 URL 完全自包含。在这种情况下，攻击者可能不需要使用外部站点，并且可以直接向受害者提供易受攻击域上的恶意 URL 。在前面的示例中，如果可以使用 GET 方法执行更改电子邮件地址的请求，则自包含的攻击如下所示：
```
<img src="https://vulnerable-website.com/email/change?email=pwned@evil-user.net">
```


## 防御 CSRF 攻击

防御 `CSRF` 攻击最有效的方法就是在相关请求中使用 `CSRF token` ，此 `token` 应该是：
- 不可预测的，具有高熵的
- 绑定到用户的会话中
- 在相关操作执行前，严格验证每种情况

可与 `CSRF token` 一起使用的附加防御措施是 `SameSite cookies` 。


## 常见的 CSRF 漏洞

最有趣的 `CSRF` 漏洞产生是因为对 `CSRF token` 的验证有问题。

在前面的示例中，假设应用程序在更改用户密码的请求中需要包含一个 `CSRF token` ：
```
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 68
Cookie: session=2yQIDcpia41WrATfjPqvm9tOkDvkMvLm

csrf=WfF1szMUHhiokx9AHFply5L2xAOfjRkE&email=wiener@normal-user.com
```

这看上去好像可以防御 `CSRF` 攻击，因为它打破了 `CSRF` 需要的必要条件：应用程序不再仅仅依赖 cookie 进行会话处理，并且请求也包含攻击者无法确定其值的参数。然而，仍然有多种方法可以破坏防御，这意味着应用程序仍然容易受到 `CSRF` 的攻击。

### CSRF token 的验证依赖于请求方法

某些应用程序在请求使用 POST 方法时正确验证 token ，但在使用 GET 方法时跳过了验证。

在这种情况下，攻击者可以切换到 GET 方法来绕过验证并发起 `CSRF` 攻击：
```
GET /email/change?email=pwned@evil-user.net HTTP/1.1
Host: vulnerable-website.com
Cookie: session=2yQIDcpia41WrATfjPqvm9tOkDvkMvLm
```

### CSRF token 的验证依赖于 token 是否存在

某些应用程序在 token 存在时正确地验证它，但是如果 token 不存在，则跳过验证。

在这种情况下，攻击者可以删除包含 token 的整个参数，从而绕过验证并发起 `CSRF` 攻击：
```
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 25
Cookie: session=2yQIDcpia41WrATfjPqvm9tOkDvkMvLm

email=pwned@evil-user.net
```

### CSRF token 未绑定到用户会话

有些应用程序不验证 token 是否与发出请求的用户属于同一会话。相反，应用程序维护一个已发出的 token 的全局池，并接受该池中出现的任何 token 。

在这种情况下，攻击者可以使用自己的帐户登录到应用程序，获取有效 token ，然后在 `CSRF` 攻击中使用自己的 token 。


### CSRF token 被绑定到非会话 cookie

在上述漏洞的变体中，有些应用程序确实将 `CSRF token` 绑定到了 cookie，但与用于跟踪会话的同一个 cookie 不绑定。当应用程序使用两个不同的框架时，很容易发生这种情况，一个用于会话处理，另一个用于 `CSRF` 保护，这两个框架没有集成在一起：
```
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 68
Cookie: session=pSJYSScWKpmC60LpFOAHKixuFuM4uXWF; csrfKey=rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv

csrf=RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY&email=wiener@normal-user.com
```

这种情况很难利用，但仍然存在漏洞。如果网站包含任何允许攻击者在受害者浏览器中设置 cookie 的行为，则可能发生攻击。攻击者可以使用自己的帐户登录到应用程序，获取有效的 token 和关联的 cookie ，利用 cookie 设置行为将其 cookie 放入受害者的浏览器中，并在 `CSRF` 攻击中向受害者提供 token 。

注意：cookie 设置行为甚至不必与 `CSRF` 漏洞存在于同一 Web 应用程序中。如果所控制的 cookie 具有适当的范围，则可以利用同一总体 DNS 域中的任何其他应用程序在目标应用程序中设置 cookie 。例如，`staging.demo.normal-website.com` 域上的 cookie 设置函数可以放置提交到 `secure.normal-website.com` 上的 cookie 。

### CSRF token 仅要求与 cookie 中的相同

在上述漏洞的进一步变体中，一些应用程序不维护已发出 token 的任何服务端记录，而是在 cookie 和请求参数中复制每个 token 。在验证后续请求时，应用程序只需验证在请求参数中提交的 token 是否与在 cookie 中提交的值匹配。这有时被称为针对 `CSRF` 的“双重提交”防御，之所以被提倡，是因为它易于实现，并且避免了对任何服务端状态的需要：
```
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 68
Cookie: session=1DQGdzYbOJQzLP7460tfyiv3do7MjyPw; csrf=R8ov2YBfTYmzFyjit8o2hKBuoIjXXVpa

csrf=R8ov2YBfTYmzFyjit8o2hKBuoIjXXVpa&email=wiener@normal-user.com
```

在这种情况下，如果网站包含任何 cookie 设置功能，攻击者可以再次执行 `CSRF` 攻击。在这里，攻击者不需要获得自己的有效 token 。他们只需发明一个 token ，利用 cookie 设置行为将 cookie 放入受害者的浏览器中，并在 `CSRF` 攻击中向受害者提供此 token 。


## 基于 Referer 的 CSRF 防御

除了使用 `CSRF token` 进行防御之外，有些应用程序使用 HTTP `Referer` 头去防御 `CSRF` 攻击，通常是验证请求来自应用程序自己的域名。这种方法通常不太有效，而且经常会被绕过。

注意：HTTP `Referer` 头是一个可选的请求头，它包含链接到所请求资源的网页的 URL 。通常，当用户触发 HTTP 请求时，比如单击链接或提交表单，浏览器会自动添加它。然而存在各种方法，允许链接页面保留或修改 `Referer` 头的值。这通常是出于隐私考虑。


### Referer 的验证依赖于其是否存在

某些应用程序当请求中有 `Referer` 头时会验证它，但是如果没有的话，则跳过验证。

在这种情况下，攻击者可以精心设计其 `CSRF` 攻击，使受害用户的浏览器在请求中丢弃 `Referer` 头。实现这一点有多种方法，但最简单的是在托管 `CSRF` 攻击的 HTML 页面中使用 META 标记：
```
<meta name="referrer" content="never">
```

### Referer 的验证可以被规避

某些应用程序以一种可以被绕过的方式验证 `Referer` 头。例如，如果应用程序只是验证 `Referer` 是否包含自己的域名，那么攻击者可以将所需的值放在 URL 的其他位置：
```
http://attacker-website.com/csrf-attack?vulnerable-website.com
```

如果应用程序验证 `Referer` 中的域以预期值开头，那么攻击者可以将其作为自己域的子域：
```
http://vulnerable-website.com.attacker-website.com/csrf-attack
```
