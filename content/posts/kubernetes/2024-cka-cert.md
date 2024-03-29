+++
draft = false
date = 2024-01-27T23:23:16+08:00
title = "我的 2024 年 CKA 认证两天速通攻略"
description = "我的 2024 年 CKA 认证两天速通攻略"
slug = ""
authors = []
tags = ["Kubernetes", "CKA"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## 背景说明

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/cka-exam.jpeg)

如上图所示，本人于 2024 年 1 月 22 号晚上 11 点进行了 CKA 的认证考试，并以 95 分（满分100）顺利通过拿证。本文将会介绍我的 CKA 考试心得和速通攻略。

## CKA 认证

官方介绍：

`CKA`( Certified Kubernetes Administrator) 认证考试可确保 Kubernetes 管理人员在从业时具备应有的技能、知识和能力。

已获得认证的 K8s 管理员具备了进行基本安装以及配置和管理生产级 Kubernetes 集群的能力。他们将了解 Kubernetes 网络、存储、安全、维护、日志记录和监控、应用生命周期、故障排除、API对象原语等关键概念，并能够为最终用户建立基本的用例。

### CKA 考试大纲

参考[官方考试大纲](https://training.linuxfoundation.cn/certificates/1)：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/CKA考试大纲.png)

大纲看着很唬人，但其实考试题目非常简单，不要被吓到。

### CKA 准备和攻略

如果你平时就接触 k8s 或者对几个核心的资源对象有基本的了解这就足够了。如果你完全什么都不懂，那也没关系，直接刷考试真题然后死记硬背也能考过，因为考试真的很简单。

考试时最好提前半个小时进入，点击考试之后会先下载一个叫 PSI 的独立软件（注意考试已经不是浏览器环境了，是独立的 APP 环境），PSI 会检查你的电脑，还会有一些权限要求，比如开启摄像头录像，以及只能有一个显示器，我用笔记本电脑外接了一台显示器结果检测不通过，由于需要摄像头，所以我笔记本电脑的屏幕没法关闭，又没有准备独立的外接摄像头，因此会检测到两台显示器然后无法进入考试，所以最后我只能放弃外接显示器直接用笔记本电脑进行考试（由于我的笔记本电脑屏幕只有13英寸，结果考试体验不太好，非常影响翻文档的效率，因此我建议你还是准备个 15 英寸以上的大屏幕）。

至于攻略，你只需要做以下两件事：
- 去 https://killercoda.com/sachin/course/CKA 刷题，玩 k8s 的一定要收藏这个网站，各种模拟环境让你不用在自己电脑或者服务器上安装 k8s 就能玩。
- 刷考试真题（几乎没变过，总是那 17 道题）。

至于购买了考试资格之后的模拟考试，其实参考意义不大，我的建议很直接，不用做模拟考（既然是速通就不要在跟考试相关性不高的地方浪费时间）。

### CKA 2024 年考试真题

我本来想把每道题都写出来的，结果发现这 17 道题几年了就几乎没变过，而且已经有人写过了真题和解答，所以这里我直接把参考的文档列出来（跟我考试时做的题简直一摸一样）：
- https://www.cnblogs.com/even160941/p/17710997.html
- https://zhuanlan.zhihu.com/p/675819358
- https://blog.csdn.net/u014481728/article/details/133421594

想要速通就训练这些真题就够了。

## 总结

就考试而言，CKA 真的非常简单，如果你平时就接触 k8s，那像我一样用两天时间刷一下题就能速通。如果你完全不懂，多花几天时间死记硬背也能躺过（如果你顺利通过考试且觉得本文对你有帮助，欢迎你回来给本文点个赞）。

最后，考证虽然简单且有技巧，但还是希望读者能够脚踏实地、认真学习并掌握相关知识。你不一定要上公有云，但一定要上云原生这朵云。