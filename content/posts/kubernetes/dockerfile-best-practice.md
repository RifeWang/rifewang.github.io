+++
draft = false
date = 2019-07-10T16:32:36+08:00
title = "Dockerfile 最佳实践"
description = "Dockerfile 最佳实践"
slug = ""
authors = []
tags = ["Docker"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

Dockerfile 是用来构建 docker 镜像的配置文件，语法简单上手容易，你可以很轻松的就编写一个能正常使用的 Dockerfile ，但是它很有可能还不够好，本文将会从细节上介绍一些 tips 助你实现最佳实践。

--------------------------

1、注意构建顺序

```diff
FROM debian
- COPY ../app
RUN apt-get update
RUN apt-get -y install cron vim ssh
+ COPY ../app
```

上例中第二步一旦本地文件发生了变化将会导致包括此后步骤的缓存全部失效，必须重新构建，尤其是在开发环境，这将会增加你构建镜像的耗时。构建步骤的排序很重要，改变小的步骤放前面，改变大的步骤放后面，这有助于你优化使用缓存加速整个构建过程。


2、使用更精确的 COPY

只 copy 真正需要的文件，比如 node_modules 或者其它一些对于构建镜像毫无作用的文件一定要忽略掉（写入 .dockerignore 文件），这些无用的文件百害而无一利。


3、合并指令

```
- RUN apt-get update
- RUN apt-get -y install cron vim ssh
+ RUN apt-get update \
   && apt-get -y install cron vim ssh
```

像这种 apt-get 升级和安装分为两个步骤毫无必要，反之统一为一个步骤更有利于缓存。你如果仔细观察各种官方镜像的 Dockerfile 是怎么写的，你肯定会发现他们单条 RUN 指令的内容相当的冗长也不会拆分，这样写是有道理的。



4、移除不必要的依赖

```
- RUN apt-get update \
   && apt-get -y install cron vim ssh
+ RUN apt-get update \
   && apt-get -y install --no-install-recommends cron
```

只安装必须的依赖，某些 debug 的工具不要在构建的时候安装，首先线上 debug 的频率应该是很低的，其次真的要用的时候另外再安装就好了。另外 apt-get 这种包管理器可能会多安装一些额外推荐的东西，加上 `--no-install-recommends` 不要安装它们，如果某些工具是需要的则必须显示声明。


5、移除包管理器的缓存

```
RUN apt-get update \
   && apt-get -y install --no-install-recommends cron \
+  && rm -rf /var/lib/apt/lists/*
```

包管理器有它自己的缓存，记得删除这些文件。做这么多的目的其实就是精简镜像的大小（一个镜像几百M 上G的实在是有太多不需要的垃圾内容了），镜像越小部署起来越快。



6、使用官方镜像

比如，你需要一个 node.js 环境，你可以拉取一个 linux 基础镜像，然后自己一步一步安装，但是这样做毫无必要，你应该直接选择 node.js 官方的基础镜像，你要知道官方镜像一定是做了很多优化的。


7、使用清晰的 tag 标记

不要使用 FROM node:latest 这种，latest 标记鬼知道具体指向了哪个版本。


8、寻找合适大小的镜像

```
12.6.0-stretch  349MB
12.6.0-slim     56MB
12.6.0-alpine   27MB
```

例如上面这三个基础镜像都是相同的 node 12.6.0 版本，但是镜像大小差别却很大，因为底层的系统是可以被裁剪的，镜像越小越好，但是要注意由于系统被裁剪可能出现兼容性问题。


9、在一致的环境中从源码构建


10、安装依赖

这里安装的依赖指的不是系统依赖，而是应用程序的依赖，在单独的步骤中进行。


11、多阶段构建

例如 go 语言，打包编译的操作是需要安装相关环境和工具的，但是运行的时候并不需要，这时候我们就可以使用多阶段构建。

```dockerfile
FROM golang:1.12 AS builder
WORKDIR /app
COPY ./ ./
RUN go build -o myapp test.go

FROM debian:stable-slim
COPY --from=builder /app/myapp /
CMD ["./myapp"]
```

如上所示，我们在第一阶段的构建拉取了完整的 go 环境，然后打包编译生成二进制可执行文件，在第二阶段则重新构造一个新的环境并选用精简的 debian 基础镜像，只把第一阶段编译好的可执行文件复制进来，而其它不需要的东西通通抛弃掉了，这样我们最终生成的镜像是非常小而精的。（多阶段构建要求 docker 版本 17.05 以上）


最后，我们所使用的语言和对环境的要求千差万别，注意不要生搬硬套，只有适合自己的才是最好的，希望本文所述的这些细节对你有所帮助。