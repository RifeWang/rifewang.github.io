+++
draft = false
date = 2019-04-11T14:49:58+08:00
title = "流平台 Kafka"
description = "流平台 Kafka"
slug = ""
authors = []
tags = ["MQ", "Kafka"]
categories = ["Middleware"]
externalLink = ""
series = []
disableComments = true
+++

## 简介

Kafka 作为一个分布式的流平台，正在大数据相关领域得到越来越广泛的应用，本文将会介绍 kafka 的相关内容。


流平台如 kafka 具备三大关键能力：

- 发布和订阅消息流，类似于消息队列。
- 以容错的方式存储消息流。
- 实时处理消息流。

kafka 通常应用于两大类应用：

- 构建实时数据流管道，以可靠的获取系统或应用之间的数据。
- 构建实时转换或响应数据流的应用程序。


kafka 作为一个消息系统，可以接受 producer 生产者投递消息，以及 consumer 消费者消费消息。

kafka 作为一个存储系统，会将所有消息以追加的方式顺序写入磁盘，这意味着消息是会被持久化的，传统消息队列中的消息一旦被消费通常都会被立即删除，而 kafka 却并不会这样做，kafka 中的消息是具有存活时间的，只有超出存活时间才会被删除，这意味着在 kafka 中能够进行消息回溯，从而实现历史消息的重新消费。

kafka 的流处理，可以持续获取输入流的数据，然后进行加工处理，最后写入到输出流。kafka 的流处理强依赖于 kafka 本身，并且只是一个类库，与当前知名的流处理框架如 spark 和 flink 还是有不小的区别和差距。

大多数使用者以及本文重点关注的也只是 kafka 的前两种能力，下面将会对此进行更加详细的介绍。


## 相关概念

kafka 中的相关概念如下图所示：

![](/images/middleware/kafka1.jpeg)


1、Producer ：生产者，投递消息。

2、Topic ：消息的逻辑分类，所有消息都必须归属于一个特定的 topic 主题。

3、Broker ：kafka 集群具有多个 broker（代理节点），一个 broker 其实就是一个 kafka 服务器。

4、Partition ：topic 只是逻辑上的概念，每个 topic 主题下的消息都会被分开存储在多个 partition 分区中，为了容错，kafka 提供了备份机制，每个 partition 可以设置多个 replication 副本。

5、Consumer ：消费者，拉取消息进行消费，每个消费者都从属于一个 consumer group 消费组。


## 消息投递

每条消息由 key、value、timestamp 构成。

消息是存储在 partition 分区上的，至于存储在哪个 partition 分区上则分以下三种情况：

1、producer 投递消息时直接指定具体的 partition 。

2、未指定 partition 并且消息中也没有 key ，那么消息将会被以轮询的方式发送到 topic 下不同的 partition 以实现负载均衡。

3、未指定 partition 但是消息中有 key ，那么将会根据 key 值计算然后发送到指定分区，相同的 key 一定是相同的 partition 。


Producer 投递消息等待响应的情况由 acks 参数确定：

1、acks = 0 ：这意味着生产者不会等待任何消息确认，也就是认为发送即成功。

2、acks = 1 ：等待 leader 写入消息成功，但不会等待 follower 的确认。这意味着 leader 确认后立马挂掉而 follower 还来不及同步消息，此时消息就会丢失。

3、acks = -1 或者 all ：不仅要 leader 确认，还需要所有 in-sync 的副本进行确认。这保证了只要有至少一个 in-sync 的副本存活，消息就不会丢失。


Leader 和 follower 指的都是 broker 对象。
每个 partition 分区都有唯一一个 broker 充当 leader，零个或多个 broker 作为 follower 。这意味着每个服务器在作为某个分区的 leader 的同时也会是其它服务器的 follower 。

消息的读写全部由 leader 处理，而 follower 只负责同步 leader 的消息。

所有正常同步的 broker 都会记录于 ISR（ In Sync Replicas ）列表中，包括 leader 本身，正常同步的状态也就是 in-sync ，如果某个服务器挂掉了或者同步进度落后太多，那么其也就不再处于 in-sync 状态，并且会从 ISR 中剔除。


## 分区存储

Topic 只是逻辑上的概念，partition 才是实际存储消息的地方，每个 topic 拥有多个 partition 分区。

![](/images/middleware/kafka2.png)

每个 partition 分区都是一个有序的不可变的记录序列，消息一定是以顺序化的方式追加写入的，也正是这种方式保证了 kafka 的高吞吐量。而每个 partition 分区中的消息都有一个 offset 偏移量作为其唯一标识。

主要注意的是单个 partition 中的消息是有序的，但是整个 topic 并不能保证消息的有序性。

消息是被持久化保存的，何时删除消息完全取决于所设置的保留期限，而与消息是否被消费没有任何关系。对于 kafka 来说，长时间存储大量数据并没有什么问题，而且也不会影响其性能。


## 消息消费

Consumer 消费消息。

![](/images/middleware/kafka3.jpeg)

每个 consumer 一定从属于一个 consumer group 消费组。

1、消息会被广播到所有订阅的 consumer group 中，不同的 group 互不影响。

2、同一个 group 中，一个 partition 分区只能同时被一个 consumer 消费，但是一个 consumer 可以同时消费多个 partition 分区，group 中的所有 consumer 一起消费所有的 partition 。

3、同一个 group 中，如果 consumer 的数量多于 partition 的数量，那么多出来的 consumer 不会做任何事情。


consumer 消费消息是需要主动向 kafka 拉取的，而不是由 kafka 推送给消费者。kafka 已经将消息进行了持久化，消费者主动拉取消息的优点就在于，消费进度完全由消费者自己掌控，其次，可以进行历史消息重新消费。

在老版本中，消费者 API 分为低级和高级两种。通过低级 API ，消费者可以指定消费特定的 partition 分区，但是对于故障转移等情况需要自己去处理。高级 API 则进行了很多底层处理并抽象了出来，消费者会被自动分配分区，并且当出现故障转移或者增减消费者或分区等情况时，会自动进行消费者再平衡，以确保消息的消费不受影响。

在新版本中，消费者 API 被重构且合并，不再分低级和高级，但消费者仍然可以自定义分区分配或者使用自动分配。

对于不同的客户端 API 使用方法需要参考各自的文档。


## 结语

kafka 具有高吞吐量、低延迟、可扩展、持久化、可容错、高并发等等特性。本文先介绍这么。