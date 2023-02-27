# web 安全之 DOM-based vulnerabilities


# DOM-based vulnerabilities

在本节中，我们将描述什么是 `DOM` ，解释对 `DOM` 数据的不安全处理是如何引入漏洞的，并建议如何在您的网站上防止基于 `DOM` 的漏洞。


## 什么是 DOM

Document Object Model（`DOM`）文档对象模型是 web 浏览器对页面上元素的层次表示。网站可以使用 JavaScript 来操作 `DOM` 的节点和对象，以及它们的属性。`DOM` 操作本身不是问题，事实上，它也是现代网站中不可或缺的一部分。然而，不安全地处理数据的 JavaScript 可能会引发各种攻击。当网站包含的 JavaScript 接受攻击者可控制的值（称为 source 源）并将其传递给一个危险函数（称为 sink 接收器）时，就会出现基于 `DOM` 的漏洞。


## 污染流漏洞

许多基于 `DOM` 的漏洞可以追溯到客户端代码在处理攻击者可以控制的数据时存在问题。


### 什么是污染流

要利用或者缓解这些漏洞，首先要熟悉 `source` 源与 `sink` 接收器之间的污染流的基本概念。

Source 源是一个 JavaScript 属性，它接受可能由攻击者控制的数据。源的一个示例是 `location.search` 属性，因为它从 query 字符串中读取输入，这对于攻击者来说比较容易控制。总之，攻击者可以控制的任何属性都是潜在的源。包括引用 URL（ `document.referrer` ）、用户的 cookies（ `document.cookie` ）和 web messages 。

Sink 接收器是存在潜在危险的 JavaScript 函数或者 DOM 对象，如果攻击者控制的数据被传递给它们，可能会导致不良后果。例如，`eval()` 函数就是一个 `sink` ，因为其把传递给它的参数当作 JavaScript 直接执行。一个 HTML sink 的示例是 `document.body.innerHTML` ，因为它可能允许攻击者注入恶意 HTML 并执行任意 JavaScript。

从根本上讲，当网站将数据从 `source` 源传递到 `sink` 接收器，且接收器随后在客户端会话的上下文中以不安全的方式处理数据时，基于 DOM 的漏洞就会出现。

最常见的 `source` 源就是 URL ，其可以通过 `location` 对象访问。攻击者可以构建一个链接，以让受害者访问易受攻击的页面，并在 URL 的 `query` 字符串和 `fragment` 部分添加有效负载。考虑以下代码：
```
goto = location.hash.slice(1)
if(goto.startsWith('https:')) {
  location = goto;
}
```

这是一个基于 DOM 的开放重定向漏洞，因为 `location.hash` 源被以不安全的方式处理。这个代码的意思是，如果 URL 的 `fragment` 部分以 `https` 开头，则提取当前 `location.hash` 的值，并设置为 `window` 的 `location` 。攻击者可以构造如下的 URL 来利用此漏洞：
```
https://www.innocent-website.com/example#https://www.evil-user.net
```

当受害者访问此 URL 时，JavaScript 就会将 `location` 设置为 `www.evil-user.net` ，也就是自动跳转到了恶意网址。这种漏洞非常容易被用来进行钓鱼攻击。


### 常见的 source 源

以下是一些可用于各种污染流漏洞的常见的 `source` 源：
```
document.URL
document.documentURI
document.URLUnencoded
document.baseURI
location
document.cookie
document.referrer
window.name
history.pushState
history.replaceState
localStorage
sessionStorage
IndexedDB (mozIndexedDB, webkitIndexedDB, msIndexedDB)
Database
```

以下数据也可以被用作污染流漏洞的 `source` 源：
- `Reflected data` 反射数据
- `Stored data` 存储数据
- `Web messages`


### 哪些 sink 接收器会导致基于 DOM 的漏洞

下面的列表提供了基于 DOM 的常见漏洞的快速概述，并提供了导致每个漏洞的 `sink` 示例。有关每个漏洞的详情请查阅本系列文章的相关部分。

| 基于 DOM 的漏洞 | sink 示例 |
| :---    |  :--- |
| DOM XSS	| document.write() |
| Open redirection 	| window.location |
| Cookie manipulation | 	document.cookie |
| JavaScript injection	| eval() |
| Document-domain manipulation | 	document.domain |
| WebSocket-URL poisoning	| WebSocket() |
| Link manipulation	| someElement.src |
| Web-message manipulation	| postMessage() |
| Ajax request-header manipulation	| setRequestHeader() |
| Local file-path manipulation	| FileReader.readAsText() |
| Client-side SQL injection	| ExecuteSql() |
| HTML5-storage manipulation	| sessionStorage.setItem() |
| Client-side XPath injection	| document.evaluate() |
| Client-side JSON injection	| JSON.parse() |
| DOM-data manipulation	| someElement.setAttribute() |
| Denial of service	| RegExp() |


### 如何防止基于 DOM 的污染流漏洞

没有一个单独的操作可以完全消除基于 `DOM` 的攻击的威胁。然而，一般来说，避免基于 `DOM` 的漏洞的最有效方法是避免允许来自任何不可信 `source` 源的数据动态更改传输到任何 `sink` 接收器的值。

如果应用程序所需的功能意味着这种行为是不可避免的，则必须在客户端代码内实施防御措施。在许多情况下，可以根据白名单来验证相关数据，仅允许已知安全的内容。在其他情况下，有必要对数据进行清理或编码。这可能是一项复杂的任务，并且取决于要插入数据的上下文，它可能需要按照适当的顺序进行 JavaScript 转义，HTML 编码和 URL 编码。

有关防止特定漏洞的措施，请参阅上表链接的相应漏洞页面。


## DOM clobbering

DOM clobbering 是一种高级技术，具体而言就是你可以将 HTML 注入到页面中，从而操作 DOM ，并最终改变网站上 JavaScript 的行为。DOM clobbering 最常见的形式是使用 anchor 元素覆盖全局变量，然后该变量将会被应用程序以不安全的方式使用，例如生成动态脚本 URL 。


