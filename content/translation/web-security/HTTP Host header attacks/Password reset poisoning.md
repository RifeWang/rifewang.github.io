+++
draft = false
date = 2021-03-06T00:00:00+08:00
title = "Password reset poisoning"
description = "Password reset poisoning"
slug = ""
authors = []
tags = []
categories = ["web security"]
externalLink = ""
series = []
disableComments = true
+++

# Password reset poisoning

密码重置中毒是一种技术，攻击者可以利用该技术来操纵易受攻击的网站，以生成指向其控制下的域的密码重置链接。这种行为可以用来窃取重置任意用户密码所需的秘密令牌，并最终危害他们的帐户。

![](https://raw.githubusercontent.com/RifeWang/images/master/web-security/password-reset-poisoning.png)


## 密码重置是如何工作的

几乎所有需要登录的网站都实现了允许用户在忘记密码时重置密码的功能。实现这个功能有好几种方法，其中一个最常见的方法是：
1. 用户输入用户名或电子邮件地址，然后提交密码重置请求。
2. 网站检查该用户是否存在，然后生成一个临时的、唯一的、高熵的 token 令牌，并在后端将该令牌与用户的帐户相关联。
3. 网站向用户发送一封包含重置密码链接的电子邮件。用户的 token 令牌作为 query 参数包含在相应的 URL 中，如 `https://normal-website.com/reset?token=0a1b2c3d4e5f6g7h8i9j`。
4. 当用户访问此 URL 时，网站会检查所提供的 token 令牌是否有效，并使用它来确定要重置的帐户。如果一切正常，用户就可以设置新密码了。最后，token 令牌被销毁。

与其他一些方法相比，这个过程足够简单并且相对安全。然而，它的安全性依赖于这样一个前提：只有目标用户才能访问他们的电子邮件收件箱，从而使用他们的 token 令牌。而密码重置中毒就是一种窃取此 token 令牌以更改其他用户密码的方法。


## 如何构造一个密码重置中毒攻击

如果发送给用户的 URL 是基于可控制的输入（例如 Host 头）动态生成的，则可以构造如下所示的密码重置中毒攻击：
1. 攻击者根据需要获取受害者的电子邮件地址或用户名，并代表受害者提交密码重置请求，但是这个请求被修改了 Host 头，以指向他们控制的域。我们假设使用的是 `evil-user.net` 。
2. 受害者收到了网站发送的真实的密码重置电子邮件，其中包含一个重置密码的链接，以及与他们的帐户相关联的 token 令牌。但是，URL 中的域名指向了攻击者的服务器：`https://evil-user.net/reset?token=0a1b2c3d4e5f6g7h8i9j` 。
3. 如果受害者点击了此链接，则密码重置的 token 令牌将被传递到攻击者的服务器。
4. 攻击者现在可以访问网站的真实 URL ，并使用盗取的受害者的 token 令牌，将用户的密码重置为自己的密码，然后就可以登录到用户的帐户了。

在真正的攻击中，攻击者可能会伪造一个假的警告通知来提高受害者点击链接的概率。

即使不能控制密码重置的链接，有时也可以使用 Host 头将 HTML 注入到敏感的电子邮件中。请注意，电子邮件客户端通常不执行 JavaScript ，但其他 HTML 注入技术如悬挂标记攻击可能仍然适用。

