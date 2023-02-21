+++
draft = false
date = 2021-03-01T00:00:00+08:00
title = "web 安全之 Directory traversal"
description = "web 安全之 Directory traversal"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++


# Directory traversal - 目录遍历

在本节中，我们将介绍什么是目录遍历，描述如何执行路径遍历攻击和绕过常见障碍，并阐明如何防止路径遍历漏洞。

![directory traversal](https://raw.githubusercontent.com/RifeWang/images/master/web-security/directory-traversal.png)


## 什么是目录遍历？

目录遍历（也称为文件路径遍历）是一个 web 安全漏洞，此漏洞使攻击者能够读取运行应用程序的服务器上的任意文件。这可能包括应用程序代码和数据、后端系统的凭据以及操作系统相关敏感文件。在某些情况下，攻击者可能能够对服务器上的任意文件进行写入，从而允许他们修改应用程序数据或行为，并最终完全控制服务器。


## 通过目录遍历读取任意文件

假设某个应用程序通过如下 HTML 加载图像：
```
<img src="/loadImage?filename=218.png">
```

这个 loadImage URL 通过 `filename` 文件名参数来返回指定文件的内容，假设图像本身存储在路径为 `/var/www/images/` 的磁盘上。应用程序基于此基准路径与请求的 `filename` 文件名返回如下路径的图像：
```
/var/www/images/218.png
```

如果该应用程序没有针对目录遍历攻击采取任何防御措施，那么攻击者可以请求类似如下 URL 从服务器的文件系统中检索任意文件：
```
https://insecure-website.com/loadImage?filename=../../../etc/passwd
```

这将导致如下路径的文件被返回：
```
/var/www/images/../../../etc/passwd
```

`../` 表示上级目录，因此这个文件其实就是：
```
/etc/passwd
```

在 Unix 操作系统上，这个文件是一个内容为该服务器上注册用户详细信息的标准文件。

在 Windows 系统上，`..\` 和 `../` 的作用相同，都表示上级目录，因此检索标准操作系统文件可以通过如下方式：
```
https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini
```


## 利用文件路径遍历漏洞的常见障碍

许多将用户输入放入文件路径的应用程序实现了某种应对路径遍历攻击的防御措施，然而这些措施却通常可以被规避。

如果应用程序从用户输入的 filename 中剥离或阻止 `..\` 目录遍历序列，那么也可以使用各种技巧绕过防御。

你可以使用从系统根目录开始的绝对路径，例如 `filename=/etc/passwd` 这样直接引用文件而不使用任何 `..\` 形式的遍历序列。

你也可以嵌套的遍历序列，例如 `....//` 或者 `....\/` ，即使内联序列被剥离，其也可以恢复为简单的遍历序列。

你还可以使用各种非标准编码，例如 `..%c0%af` 或者 `..%252f` 以绕过输入过滤器。

如果应用程序要求用户提供的文件名必须以指定的文件夹开头，例如 `/var/www/images` ，则可以使用后跟遍历序列的方式绕过，例如：
```
filename=/var/www/images/../../../etc/passwd
```

如果应用程序要求用户提供的文件名必须以指定的后缀结尾，例如 `.png` ，那么可以使用空字节在所需扩展名之前有效地终止文件路径并绕过检查：
```
filename=../../../etc/passwd%00.png
```


## 如何防御目录遍历攻击

防御文件路径遍历漏洞最有效的方式是避免将用户提供的输入直接完整地传递给文件系统 API 。许多实现此功能的应用程序部分可以重写，以更安全的方式提供相同的行为。

如果认为将用户输入传递到文件系统 API 是不可避免的，则应该同时使用以下两层防御措施：
- 应用程序对用户输入进行严格验证。理想情况下，通过白名单的形式只允许明确的指定值。如果无法满足需求，那么应该验证输入是否只包含允许的内容，例如纯字母数字字符。
- 验证用户输入后，应用程序应该将输入附加到基准目录下，并使用平台文件系统 API 规范化路径，然后验证规范化后的路径是否以基准目录开头。

下面是一个简单的 Java 代码示例，基于用户输入验证规范化路径：
```
File file = new File(BASE_DIRECTORY, userInput);
if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
    // process file
}
```