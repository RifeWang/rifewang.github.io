+++
draft = false
date = 2024-08-15T17:06:40+08:00
title = "典型系统架构的百万并发理论分析"
description = "典型系统架构的百万并发理论分析"
slug = ""
authors = []
tags = ["系统架构"]
categories = ["系统架构"]
externalLink = ""
series = []
disableComments = true
+++

## 引言

本文将会描述一个典型的系统架构，然后分析其在理论上是否能够支撑百万并发的请求。

## 典型系统架构及分析

为了降低复杂性，笔者将系统简化为了下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/arch/typical-system-arch.png)

该系统架构包含的组件有：
- 网关层：Load Balancer / API Gateway
- 服务层：HTTP Server
- 中间件：Redis、MQ
- 存储层：MySQL / PostgreSQL

忽略业务场景的复杂性，让我们依次分析各个常见的组件。

### Load Balancer

负责均衡 & API 网关：
- 负责均衡主要关注的是流量分发这一单一功能。
- API 网关涉及更广泛的功能集，包括但不限于 API 管理、安全、转换、监控等。
- 两者在具体的产品上界限比较模糊。

LB 按类型可分为：硬件负载均衡器、软件负载均衡器、云负载均衡器。

常用的开源软件 LB 包括：Nginx、HAProxy、Kong、Traefik、Envoy 等等。

常见 LB 支持的 RPS（Requests Per Second） 从几万到几十万不等，依赖于硬件和配置。在高性能硬件配置的情况下可以支持百万请求转发。而云服务商提供的 LB 宣称支持百万、千万级别。

参考资料：
- [HAProxy 单实例每秒两百万请求转发](https://www.haproxy.com/blog/haproxy-forwards-over-2-million-http-requests-per-second-on-a-single-aws-arm-instance)
- [AWS NLB 支持每秒百万请求](https://aws.amazon.com/blogs/aws/new-network-load-balancer-effortless-scaling-to-millions-of-requests-per-second/)


### HTTP Server

忽略业务处理逻辑，HTTP Server 根据编程语言框架的不同，RPS 从几万到几十万不等，甚至能达到几百万。由于无状态的特性，HTTP Server 也很容易进行横向扩展，因此支持百万并发完全没有问题。

如下图所示：
![](https://raw.githubusercontent.com/RifeWang/images/master/arch/web-frameworks.png)

参考资料：
- [Web Framework Benchmark](https://www.techempower.com/benchmarks/#section=data-r22&test=fortune)


### Redis

一般情况我们认为 Redis 单机 QPS 可以达到 10 万，根据 Redis Cluster 的线性扩展性理论（单机 QPS 为 m，那么 n 台机器的集群的总 QPS 为 m * n，官方文档说集群上限可以有 1000 个节点），Redis 集群可以支撑百万级、甚至千万 QPS。而 AWS 扩展到了单集群 5 亿 QPS。

参考资料：
- [Redis Cluster Spec](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [AWS 将 Redis 集群扩展到了 5 亿 QPS 且响应时间为微秒](https://aws.amazon.com/blogs/database/achieve-over-500-million-requests-per-second-per-cluster-with-amazon-elasticache-for-redis-7-1/)

### MQ

对于消息队列，我们把百万并发转换成其每秒接受百万条消息的能力，由于 MQ 往往是追加写入，再加上集群的支持，支撑百万并发也没有什么问题。

参考资料：
- [Kafka 在三台廉价机器上进行每秒 2 百万写入](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)

### MySQL / PostgreSQL

如下是某云服务商提供的 MySQL 5.7 性能测试数据：

![](https://raw.githubusercontent.com/RifeWang/images/master/arch/mysql5.7-benchmark.png)

从图中可以看到即使是 16C64G 并发度 128 的情况下支持的 QPS 也不到 14 万，TPS 不到 7000。另外再考虑连接数和线程的限制（MySQL 会为每条连接建立一个线程），数据库想要直接支持百万 QPS 不可能，传统关系型数据库是一个明显的瓶颈点，只能做分库、缓存、限流等措施。

参考资料：
- [MySQL 5.7 性能测试](https://cloud.tencent.com/document/product/236/68814)


### 网络

网络带宽是一个不可忽视的点。

数据中心的网络带宽常见规格有：10 Gbps、40 Gbps、100 Gbps、400 Gbps 甚至更高。

以 10 Gbps 为例，10 Gbps = 1.25 GB/s，假设每个请求大小是 1 KB，那么每秒 1 百万请求则需要 1 GB/s，可以看出来 10 Gbps 的带宽接近极限。


## 总结

本文化繁为简，以一个典型的系统架构为例，从理论上分析了各个组件的性能上限，以及对百万并发的支撑情况。对常用组件心中有数，在系统设计时才能有理有据。至于真实业务场景下的系统最大并发数，则只能由系统压测得到结果。