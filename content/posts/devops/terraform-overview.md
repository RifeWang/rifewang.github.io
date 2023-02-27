+++
draft = false
date = 2022-03-27T00:00:00+08:00
title = "Terraform: 基础设施即代码"
description = ""
slug = ""
authors = []
tags = ["CICD"]
categories = []
externalLink = ""
series = []
disableComments = true
+++

# Terraform: 基础设施即代码

## 问题

现如今有很多 IT 系统的基础设施直接使用了云厂商提供的服务，假设我们需要构建以下基础设施：
- VPC 网络
- 虚拟主机
- 负载均衡器
- 数据库
- 文件存储
- ...

那么在公有云的环境中，我们一般怎么做？

在云厂商提供的前端管理页面上手动操作吗？

这也太费劲了吧，尤其是当基础设施越来越多、越来越复杂、以及跨多个云环境的时候，这些基础设施的配置和管理便会碰到一个巨大的挑战。

## Terraform

为了解决上述问题，Terrafrom 应运而生。

使用 Terraform ，我们只需要编写简单的声明式代码，形如：
```
...

resource "alicloud_db_instance" "instance" {
  engine           = "MySQL"
  engine_version   = "5.6"
  instance_type    = "rds.mysql.s1.small"
  instance_storage = "10"
  ...
}
```
然后执行几个简单的 terraform 命令便可以轻松创建一个阿里云的数据库实例。

![terraform](https://raw.githubusercontent.com/RifeWang/images/master/terraform/terraform.png)


这就是 `Infrastructure as code` 基础设施即代码。也就是通过代码而不是手动流程来管理和配置基础设施。

正如其官方文档所述，与手动管理基础设施相比，使用 Terraform 有以下几个优势：
- Terraform 可以轻松管理多个云平台上的基础设施。
- 使用人类可读的声明式的配置语言，有助于快速编写基础设施代码。
- Terraform 的状态允许您在整个部署过程中跟踪资源更改。
- 可以对这些基础设施代码进行版本控制，从而安全地进行协作。


### Provider & Module

你也许会感到困惑，我只是简单的应用了所写的声明式代码，怎么就构建出来了基础设施，这中间发生了什么？

其实简而言之就是 terraform 在执行的过程中内部调用了基础设施平台提供的 API 。

![provider](https://raw.githubusercontent.com/RifeWang/images/master/terraform/intro-terraform-apis.png)

每个基础设施平台都会把对自身资源的操作统一封装打包成一个 provider 。provider 的概念就好像是编程语言中的一个依赖库。

在 terraform 中引用 provider ：
```
terraform {
  required_providers {
    alicloud = {
      source = "aliyun/alicloud"
      version = "1.161.0"
    }
  }
}

provider "alicloud" {
  # Configuration options
}
```


我们在写代码的时候经常会把某些可重用的部分剥离出来作为一个模块，而在 terraform 中，对基础设施的管理也是如此，我们能够把可重用的 terraform 配置组成 module 模块，我们即可以在我们 local 本地自己编写模块，也可以直接使用第三方组织好并且公开发布的 remote 模块。

![provider & module](https://raw.githubusercontent.com/RifeWang/images/master/terraform/terraform-provider-module.png)


## 最后

本文只是抛砖引玉罢了，有关 terraform 的更多内容还请参考官方文档及其它资料。