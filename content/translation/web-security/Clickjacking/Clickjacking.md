+++
draft = false
date = 2021-03-05T00:00:00+08:00
title = "web 安全之 Clickjacking"
description = "web 安全之 Clickjacking ( UI redressing )"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# Clickjacking ( UI redressing )

在本节中，我们将解释什么是 clickjacking 点击劫持，并描述常见的点击劫持攻击示例，以及讨论如何防御这些攻击。


## 什么是点击劫持

点击劫持是一种基于界面的攻击，通过诱导用户点击钓鱼网站中的被隐藏了的可操作的危险内容。

例如：某个用户被诱导访问了一个钓鱼网站（可能是点击了电子邮件中的链接），然后点击了一个赢取大奖的按钮。实际情况则是，攻击者在这个赢取大奖的按钮下面隐藏了另一个网站上向其他账户进行支付的按钮，而结果就是用户被诱骗进行了支付。这就是一个点击劫持攻击的例子。这项技术实际上就是通过 `iframe` 合并两个页面，真实操作的页面被隐藏，而诱骗用户点击的页面则显示出来。点击劫持攻击与 `CSRF` 攻击的不同之处在于，点击劫持需要用户执行某种操作，比如点击按钮，而 `CSRF` 则是在用户不知情或者没有输入的情况下伪造整个请求。

![clickjacking](https://raw.githubusercontent.com/RifeWang/images/master/web-security/clickjacking.png)

针对 `CSRF` 攻击的防御措施通常是使用 `CSRF token`（针对特定会话、一次性使用的随机数）。而点击劫持无法则通过 `CSRF token` 缓解攻击，因为目标会话是在真实网站加载的内容中建立的，并且所有请求均在域内发生。`CSRF token` 也会被放入请求中，并作为正常行为的一部分传递给服务器，与普通会话相比，差异就在于该过程发生在隐藏的 `iframe` 中。


## 如何构造一个基本的点击劫持攻击

点击劫持攻击使用 CSS 创建和操作图层。攻击者将目标网站通过 `iframe` 嵌入并隐藏。使用样式标签和参数的示例如下：
```
<head>
  <style>
    #target_website {
      position:relative;
      width:128px;
      height:128px;
      opacity:0.00001;
      z-index:2;
      }
    #decoy_website {
      position:absolute;
      width:300px;
      height:400px;
      z-index:1;
      }
  </style>
</head>
...
<body>
  <div id="decoy_website">
  ...decoy web content here...
  </div>
  <iframe id="target_website" src="https://vulnerable-website.com">
  </iframe>
</body>
```

目标网站 `iframe` 被定位在浏览器中，使用适当的宽度和高度位置值将目标动作与诱饵网站精确重叠。无论屏幕大小，浏览器类型和平台如何，绝对位置值和相对位置值均用于确保目标网站准确地与诱饵重叠。`z-index` 决定了 `iframe` 和网站图层的堆叠顺序。透明度被设置为零，因此 `iframe` 内容对用户是透明的。浏览器可能会基于 `iframe` 透明度进行阈值判断从而自动进行点击劫持保护（例如，Chrome 76 包含此行为，但 Firefox 没有），但攻击者仍然可以选择适当的透明度值，以便在不触发此保护行为的情况下获得所需的效果。


## 预填写输入表单

一些需要表单填写和提交的网站允许在提交之前使用 GET 参数预先填充表单输入。由于 GET 参数在 URL 中，那么攻击者可以直接修改目标 URL 的值，并将透明的“提交”按钮覆盖在诱饵网站上。


## Frame 拦截脚本

只要网站可以被 `frame` ，那么点击劫持就有可能发生。因此，预防性技术的基础就是限制网站 `frame` 的能力。比较常见的客户端保护措施就是使用 web 浏览器的 `frame` 拦截或清理脚本，比如浏览器的插件或扩展程序，这些脚本通常是精心设计的，以便执行以下部分或全部行为：
- 检查并强制当前窗口是主窗口或顶部窗口
- 使所有 `frame` 可见。
- 阻止点击可不见的 `frame`。
- 拦截并标记对用户的潜在点击劫持攻击。

Frame 拦截技术一般特定于浏览器和平台，且由于 HTML 的灵活性，它们通常也可以被攻击者规避。由于这些脚本也是 JavaScript ，浏览器的安全设置也可能会阻止它们的运行，甚至浏览器直接不支持 JavaScript 。攻击者也可以使用 HTML5 `iframe` 的 `sandbox` 属性去规避 frame 拦截。当 `iframe` 的 `sandbox` 设置为 `allow-forms` 或 `allow-scripts`，且 `allow-top-navigation` 被忽略时，frame 拦截脚本可能就不起作用了，因为 iframe 无法检查它是否是顶部窗口：
```
<iframe id="victim_website" src="https://victim-website.com" sandbox="allow-forms"></iframe>
```

当 `iframe` 的 `allow-forms` 和 `allow-scripts` 被设置，且 top-level 导航被禁用，这会抑制 frame 拦截行为，同时允许目标站内的功能。


## 结合使用点击劫持与 DOM XSS 攻击

到目前为止，我们把点击劫持看作是一种独立的攻击。从历史上看，点击劫持被用来执行诸如在 Facebook 页面上增加“点赞”之类的行为。然而，当点击劫持被用作另一种攻击的载体，如 DOM XSS 攻击，才能发挥其真正的破坏性。假设攻击者首先发现了 XSS 攻击的漏洞，则实施这种组合攻击就很简单了，只需要将 `iframe` 的目标 URL 结合 XSS ，以使用户点击按钮或链接，从而执行 DOM XSS 攻击。


## 多步骤点击劫持

攻击者操作目标网站的输入可能需要执行多个操作。例如，攻击者可能希望诱骗用户从零售网站购买商品，而在下单之前还需要将商品添加到购物篮中。为了实现这些操作，攻击者可能使用多个视图或 `iframe` ，这也需要相当的精确性，攻击者必须非常小心。


## 如何防御点击劫持攻击

我们在上文中已经讨论了一种浏览器端的预防机制，即 frame 拦截脚本。然而，攻击者通常也很容易绕过这种防御。因此，服务端驱动的协议被设计了出来，以限制浏览器 `iframe` 的使用并减轻点击劫持的风险。

点击劫持是一种浏览器端的行为，它的成功与否取决于浏览器的功能以及是否遵守现行 web 标准和最佳实践。服务端的防御措施就是定义 `iframe` 组件使用的约束，然而，其实现仍然取决于浏览器是否遵守并强制执行这些约束。服务端针对点击劫持的两种保护机制分别是 `X-Frame-Options` 和 `Content Security Policy` 。


## X-Frame-Options

`X-Frame-Options` 最初由 IE8 作为非官方的响应头引入，随后也在其他浏览器中被迅速采用。`X-Frame-Options` 头为网站所有者提供了对 `iframe` 使用的控制（就是说第三方网站不能随意的使用 `iframe` 嵌入你控制的网站），比如你可以使用 `deny` 直接拒绝所有 `iframe` 引用你的网站：
```
X-Frame-Options: deny
```

或者使用 `sameorigin` 限制为只有同源网站可以引用：
```
X-Frame-Options: sameorigin
```

或者使用 `allow-from` 指定白名单：
```
X-Frame-Options: allow-from https://normal-website.com
```

`X-Frame-Options` 在不同浏览器中的实现并不一致（比如，Chrome 76 或 Safari 12 不支持 `allow-from`）。然而，作为多层防御策略中的一部分，其与 `Content Security Policy` 结合使用时，可以有效地防止点击劫持攻击。


## Content Security Policy

`Content Security Policy` (`CSP`) 内容安全策略是一种检测和预防机制，可以缓解 XSS 和点击劫持等攻击。`CSP` 通常是由 web 服务作为响应头返回，格式为：
```
Content-Security-Policy: policy
```

其中的 `policy` 是一个由分号分隔的策略指令字符串。`CSP` 向客户端浏览器提供有关允许的 Web 资源来源的信息，浏览器可以将这些资源应用于检测和拦截恶意行为。

有关点击劫持的防御，建议在 `Content-Security-Policy` 中增加 `frame-ancestors` 策略。

- `frame-ancestors 'none'` 类似于 `X-Frame-Options: deny` ，表示拒绝所有 iframe 引用。
- `frame-ancestors 'self'` 类似于 `X-Frame-Options: sameorigin` ，表示只允许同源引用。

示例：
```
Content-Security-Policy: frame-ancestors 'self';
```

或者指定网站白名单：
```
Content-Security-Policy: frame-ancestors normal-website.com;
```

为了有效地防御点击劫持和 XSS 攻击，`CSP` 需要进行仔细的开发、实施和测试，并且应该作为多层防御策略中的一部分使用。