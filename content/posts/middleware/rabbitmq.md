+++
draft = false
date = 2018-02-27T18:53:07+08:00
title = "RabbitMQ 入门教程及示例"
description = "RabbitMQ 入门教程及示例"
slug = ""
authors = []
tags = ["MQ", "Node.js"]
categories = ["RabbitMQ"]
externalLink = ""
series = []
disableComments = true
+++

## 一

消息中间件 MQ（也称消息队列）的基本功能是传递和转发消息，其最重要的作用是能够解耦业务及系统架构，可以说是一个系统发展壮大到一定阶段绕不开的东西。

而 RabbitMQ 是对 AMQP（高级消息队列协议）的实现，成熟可靠并且开源，本系列文章将会讲述如何在 node 中入门这一利器。

### RabbitMQ 概述

先来简单的了解一下 RabbitMQ 相关的基本概念：

![](/images/middleware/rabbitmq1-1.jpeg)

Producer ：生产者，生成消息并把消息发送给 RabbitMQ 。

Consumer ：消费者，从 RabbitMQ 中接收消息。

Exchange ：交换器，具有路由的作用，将生产者传递的消息根据不同的路由规则传递到对应的队列中。交换器具有四种不同的类型，每种类型对应不同的路由规则。

Queue ：队列，实际存储消息的地方，消费者通过订阅队列来获取队列中的消息。

Binding ：绑定交换器和队列，只有绑定后消息才能被交换器分发到具体的队列中，用一个字符串来代表 Binding Key 。



消息是如何由生产者传递到消费者：

1. 生产者 Producer 生成消息 msg ，并指定这条消息的路由键 Routing Key ，然后将消息传递给交换器 Exchange 。

2. 交换器 Exchange 接收到消息后根据 Exchange Type 也就是交换器类型以及交换器和队列的 Binding 绑定关系来判断路由规则并分发消息到具体的队列 Queue 中。

3. 消费者 Consumer 通过订阅具体的队列，一旦队列接收到消息便会将其传递给消费者。

这里的 Routing Key 和 Binding 我是按照自己的理解解释的，与某些参考资料是有出入的，读者理解就好。



当然完成上述三个步骤还缺少两个关键的东西：

- Connection ：连接，不论生产者还是消费者想要使用 RabbitMQ 都必须首先建立到 RabbitMQ 的 TCP 连接。

- Channel ：信道，建立完 TCP 连接后还必须建立一个信道，消息都是在信道中传递和操作的。

![](/images/middleware/rabbitmq1-2.jpeg)

上图形象的展示了连接和信道之间的关系，一个连接中可以建立多个信道，而且每个信道之间都是完全隔离的，同时我们需要记住的是创建和销毁 TCP 连接是很消耗资源的，而信道则不是，所以能够通过创建多个信道来隔离环境的不要通过创建多个连接。


### 交换器类型

交换器具有路由分发消息的作用，其有四种不同的类型，每种类型对应不同的路由规则：
- fanout ：广播，将消息传递给所有该交换器绑定的队列。
- direct ：直连，将消息传递给 Routing Key 与 Binding Key完全一致的队列中，可以有多个队列。
- topic ：模糊匹配，Binding Key 是一个可以用符号 . 分隔单词的字符串，模糊匹配下，符号 * 用于匹配任意一个单词，符号  # 用于匹配零个或多个单词。
- headers ：这个比较特殊，是根据消息中具体内容的 header 属性来作为路由规则的，这种类型对资源消耗太大，一般很少使用，前面三种类型就够了。


---

## 二

### 流程

我们先来了解一下 RabbitMQ 的一般使用流程。

1. 建立到 RabbitMQ 的连接。
2. 创建信道。
3. 声明交换器。
4. 声明队列。
5. 绑定交换器和队列。
6. 消息操作。生产者：生成并发布消息；消费者：订阅并消费消息。
7. 关闭信道。
8. 关闭连接。

不论是生产者投递消息，还是消费者接受消息一般都遵循以上步骤，但针对具体的情况仍会有调整，比如声明交换器、声明队列、绑定交换器和队列，我们只需要在生产者或消费者其中之一进行，甚至隔离出来独立维护，只要保证在发布或消费消息之前交换器、队列、绑定等是有效的即可。


### Hello World 示例

第一个示例，实现基本的投递和接收消息。

生产者投递消息（send.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();     //建立信道
        const queueName = 'hello';
        const msg = 'Hello world';
        await ch.assertQueue(queueName, { durable: false });   //声明队列，durable：false 不对队列持久化
        ch.sendToQueue(queueName, new Buffer(msg));   //发送消息
        console.log(' [x] Sent %s', msg);
        await ch.close();   //关闭信道
        await conn.close();  //关闭连接
    } catch (error) {
        console.log(error);
    }
})()
```


消费者接收消息（receive.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();  //建立信道
        const queueName = 'hello';
        await ch.assertQueue(queueName, { durable: false });  //声明队列，durable：false 不对队列持久化
        console.log(" [*] Waiting for messages in queue: %s. To exit press CTRL+C", queueName);
        ch.consume(queueName, msg => {   //订阅队列接受消息
            console.log(" [x] Received %s", msg.content.toString());
        }, { noAck: true });   // noAck：true 不进行确认接受应答
    } catch (error) {
        console.log(error);
    }
})()
```


对比上述流程，你会发现为什么没有交换器 Exchange 存在的身影呢？这是因为 RabbitMQ 存在一个默认交换器，类型为 direct （直连），每个新建的队列会自动绑定到默认交换器上，并且以队列的名称作为绑定路由规则。

声明队列时，同一个队列其属性前后相同时，重复声明不会有任何影响，反之其属性前后不相同时，重复声明会抛出一个错误，这种情况要注意不得重复声明，当然如果这个队列被声明有效了也不需要再次声明。

从上例中我们也了解到了队列的一个属性 durable，这个属性表明是否对队列进行持久化，也就是保存到磁盘上，一旦 RabbitMQ 服务器重启，持久化的队列可以被重新恢复。

消费者 consume 订阅接收消息时使用了另一个属性 noAck，这个属性表明消费者在接收到消息后是否需要向 RabbitMQ 服务器确认收到该消息。与之相对的是发后即忘模式，也就是 RabbitMQ 服务器向消费者发送完消息后即认为成功，无需等待消费者确认接收应答，这种模式吞吐量更高，但可靠性显然不如确认应答模式，而确认应答模式，我们需要注意的是， RabbitMQ 服务器若没有接收到 ack 确认会一直将该消息保存，如果消费者挂了就会造成消息持续堆叠不断占用内存的情况，极端情况下资源过载会造成 RabbitMQ 服务器重启，同时未被 ack 确认的消息会被尝试重新发送给消费者。



### Work queues

第二个示例，向多个消费者分发投递消息。

生产者投递消息（new_task.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();   //建立信道
        const queueName = 'task_queue';
        await ch.assertQueue(queueName, { durable: true });   //声明队列，durable：true 持久化队列

        for (let i = 1; i < 10; i++) {   //生成 9 条信息并在尾部添加小数点
            let msg = i.toString().padEnd(i+1, '.');
            ch.sendToQueue(queueName, new Buffer(msg), { persistent: true });   //发送消息，persistent：true 将消息持久化
            console.log(" [x] Sent '%s'", msg);
        }

        await ch.close();     //关闭信道
        await conn.close();    //关闭连接
    } catch (error) {
        console.log(error);
    }
})()
```


消费者接收消息（worker.js）：
```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();    //建立信道
        const queueName = 'task_queue';
        await ch.assertQueue(queueName, { durable: true });   //声明队列
        ch.prefetch(1);  //每次接收不超过指定数量的消息
        console.log(" [*] Waiting for messages in %s. To exit press CTRL+C", queueName);

        ch.consume(queueName, msg => {
            const secs = msg.content.toString().split('.').length - 1;
            console.log(" [x] Received %s", msg.content.toString());
            setTimeout(() => {   //根据小数点的个数设置延时时长
                console.log(" [x] Done");
                ch.ack(msg);    //确认接收消息
            }, secs * 1000);
        }, { noAck: false });   //对消息需要接收确认

    } catch (error) {
        console.log(error);
    }
})()
```


我们在 shell 中运行多个 worker.js 会发现消息被一个一个分发到了不同的 worker 消费者，且同一条消息不会被重复发送给多个 worker 。


在这个示例中，我们对队列进行了持久化，并且在消费端使用了 ack 确认接收消息。发送消息时，我们使用了 persistent 属性，这个属性表明是否将消息持久化。另外，对消费者而言，还使用了 ch.prefetch() 方法，这个方法表明该消费者每次最多接收的消息数量，这样做是因为某些情况下消费消息是一个很耗时的业务操作，某些 worker 可能处于繁忙状态，而另外一些 worker 则很空闲，通过 prefetch 和 ack 其实是实现了类似于负载均衡的功能，也就是将消息分发给空闲的 worker 消费。

---

## 三

我们再来回顾一遍 RabbitMQ 的一般使用流程：

1. 建立到 RabbitMQ 的连接。
2. 创建信道。
3. 声明交换器。
4. 声明队列。
5. 绑定交换器和队列。
6. 消息操作。生产者：生成并发布消息；消费者：订阅并消费消息。
7. 关闭信道。
8. 关闭连接。


交换器 Exchange 的四种类型：

1. fanout：广播，将消息传递给所有该交换器绑定的队列。
2. direct ：直连，将消息传递给 Routing Key 与 Binding Key完全一致的队列中，可以有多个队列。
3. topic ：模糊匹配，Binding Key 是一个可以用符号 . 分隔单词的字符串，模糊匹配下，符号 * 用于匹配任意一个单词，符号  # 用于匹配零个或多个单词。
4. headers ：根据消息中具体内容的 header 属性来作为路由规则的，这种类型对资源消耗太大且很少使用，本节不对此类型进行讲述。


### Publish/Subscribe

此示例重点关注交换器 Exchange 的 fanout 类型。

消费者接收消息（receive_log.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();   //建立信道

        const ex = 'logs';
        await ch.assertExchange(ex, 'fanout', { durable: false });   //声明交换器
        const q = await ch.assertQueue('', { exclusive: true });   //声明队列，临时队列即用即删
        console.log(" [*] Waiting for messages in %s. To exit press CTRL+C", q.queue);
        await ch.bindQueue(q.queue, ex, '');   //绑定交换器和队列，参数：队列名、交换器名、绑定键值

        ch.consume(q.queue, msg => {   //订阅队列接收消息
            console.log(" [x] %s", msg.content.toString());
        }, { noAck: true });

    } catch (error) {
        console.log(error);
    }
})()
```

生产者投递消息（emit_log.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();  //建立信道
        const ex = 'logs';
        const msg = process.argv.slice(2).join(' ') || 'Hello World!';

        await ch.assertExchange(ex, 'fanout', { durable: false });   //声明 exchange ，类型为 fanout ，不持久化
        ch.publish(ex, '', new Buffer(msg));   //发送消息，fanout 类型无需指定 routing key
        console.log(" [x] Sent %s", msg);

        await ch.close();   //关闭信道
        await conn.close();  //关闭连接
    } catch (error) {
        console.log(error);
    }
})();
```

fanout 类型的交换器会直接将消息广播到所有与其绑定的队列，所以绑定交换器与队列时无需指定 binding key （空字符串），投递消息时也无需指定 routing key （空字符串）。

交换器与队列一样具有 durable 属性，此属性表示是否对交换器进行持久化，也就是保存到磁盘上，一旦 RabbitMQ 服务器重启，持久化的交换器可以被重新恢复。

这里在声明队列时，我们使用的是一种临时的队列，我们无需指定该队列的名称，RabbitMQ 会自动为其生成一个随机的名称，同时 exclusive 属性表明该队列是否只会被当前连接使用，也就是说连接一旦关闭则此队列也会被删除。

### Routing

此示例重点关注交换器 Exchange 的 direct 类型。

消费者接收消息（receive_log_direct.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const args = process.argv.slice(2);
        if (args.length == 0) {
            console.log("Usage: receive_logs_direct.js [info] [warning] [error]");
            process.exit(1);
        }

        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();    //建立信道

        const ex = 'direct_logs';
        await ch.assertExchange(ex, 'direct', { durable: false });    //声明交换器
        const q = await ch.assertQueue('', { exclusive: true });    //声明队列

        console.log(' [*] Waiting for logs. To exit press CTRL+C');
        args.forEach(async severity => {
            await ch.bindQueue(q.queue, ex, severity);  //绑定交换器和队列
        });

        ch.consume(q.queue, msg => {   //订阅队列接收消息
            console.log(" [x] %s: '%s'", msg.fields.routingKey, msg.content.toString());
        }, { noAck: true });

    } catch (error) {
        console.log(error);
    }
})()
```

生产者投递消息（emit_log_direct.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();  //建立信道
        const ex = 'direct_logs';

        const args = process.argv.slice(2);
        const msg = args.slice(1).join(' ') || 'Hello World!';
        const severity = (args.length > 0) ? args[0] : 'info';

        await ch.assertExchange(ex, 'direct', { durable: false });  //声明 exchange ，类型为 direct
        ch.publish(ex, severity, new Buffer(msg));   //发送消息，参数：交换器、路由键、消息内容
        console.log(" [x] Sent %s: '%s'", severity, msg);

        await ch.close();   //关闭信道
        await conn.close();  //关闭连接
    } catch (error) {
        console.log(error);
    }
})();
```

交换器为 direct 类型，路由规则是 routing key 与 binding key 完全一致，这就是说与上例 fanout 类型不同的是，我们必须指定绑定交换器和队列的 binding key ，投递消息时也需要指定路由的 routing key 。其余地方基本一致。


### Topics

此示例重点关注交换器 Exchange 的 topic 类型。

消费者接收消息（receive_log_topic.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const args = process.argv.slice(2);
        if (args.length == 0) {
            console.log("Usage: receive_logs_topic.js <facility>.<severity>");
            process.exit(1);
        }

        const conn = await amqp.connect('amqp://localhost');    //建立连接
        const ch = await conn.createChannel();     //建立信道

        const ex = 'topic_logs';
        await ch.assertExchange(ex, 'topic', { durable: false });   //声明交换器
        const q = await ch.assertQueue('', { exclusive: true });    //声明队列
        console.log(' [*] Waiting for logs. To exit press CTRL+C');

        args.forEach(async key => {
            await ch.bindQueue(q.queue, ex, key);   //绑定交换器和队列
        });

        ch.consume(q.queue, msg => {    //订阅队列接收消息
            console.log(" [x] %s:'%s'", msg.fields.routingKey, msg.content.toString());
        }, { noAck: true });
    } catch (error) {
        console.log(error);
    }
})()
```

生产者投递消息（emit_log_topic.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();   //建立信道

        const ex = 'topic_logs';
        const args = process.argv.slice(2);
        const key = (args.length > 0) ? args[0] : 'anonymous.info';
        const msg = args.slice(1).join(' ') || 'Hello World!';

        await ch.assertExchange(ex, 'topic', { durable: false });  //声明交换器
        ch.publish(ex, key, new Buffer(msg));   //发送消息，指定 routing key
        console.log(" [x] Sent %s: '%s'", key, msg);

        await ch.close();   //关闭信道
        await conn.close();  //关闭连接
    } catch (error) {
        console.log(error);
    }
})()
```

交换器的 topic 类型，只需注意模糊匹配的规则即可，绑定交换器和队列的 binding key 以符号 . 将字符串分隔为不同的单词（不一定是真实的单词，理解为一个部分就行了），符号 * 用于匹配任意一个单词，符号  # 用于匹配零个或多个单词。

其实你会发现本节三个示例中的大部分地方都是类似的，唯一不同的地方就是不同的交换器类型需要对 binding key 和 routing key 进行不同的处理。通过本节了解了不同的交换器类型，有助于你在此基础上进行具体的路由规则设计。

---

## 四

### RPC

RPC 是什么？Remote Procedure Call，远程过程调用，比如某个服务器调用另一个远程服务器上的函数或方法获取其结果，当然这种类似需求毫无疑问是可以用我们熟悉的 REST 来实现的。

使用 RabbitMQ 如何实现 RPC 的功能：

![](/images/middleware/rabbitmq4-1.png)

如上图所示，客户端发起请求到一个 rpc 队列，并指定一个 correlationId 作为该请求的唯一标识，且通过 reply_to 指定一个 callback 队列接收请求处理结果（这里的 callback 并不是指 node 中的回掉函数，注意区别）。服务端通过订阅指定的 rpc 队列接收到请求然后进行处理，处理完之后将结果发送到 reply_to 指定的 callback 队列中，客户端通过订阅 callback 队列获取请求结果，并通过 correlationId 对应不同的请求。

客户端示例（rpc_client.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const args = process.argv.slice(2);
        if (args.length === 0) {
            console.log("Usage: rpc_client.js num");
            process.exit(1);
        }

        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();  //建立信道
        const q = await ch.assertQueue('', { exclusive: true });    //声明一个临时队列作为 callback 接收结果

        const corr = generateUuid();
        const num = parseInt(args[0]);
        console.log(' [x] Requesting fib(%d)', num);

        ch.consume(q.queue, async (msg) => {   //订阅 callback 队列接收 RPC 结果
            if (msg.properties.correlationId === corr) {    //根据 correlationId 判断是否为请求的结果
                console.log(' [.] Got %s', msg.content.toString());
                await ch.close();   //关闭信道
                await conn.close();    //关闭连接
            }
        }, { noAck: true });

        ch.sendToQueue('rpc_queue',       //发送 RPC 请求
            new Buffer(num.toString()),
            {
                correlationId: corr,      // correlationId 将 RPC 结果与对应的请求关联，replyTo 指定结果返回的队列
                replyTo: q.queue
            }
        );
    } catch (error) {
        console.log(error);
    }
})();

function generateUuid() {     //唯一标识
    return Math.random().toString() + Math.random().toString() + Math.random().toString();
}
```

服务端示例（rpc_server.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();   //建立信道
        const q = 'rpc_queue';

        await ch.assertQueue(q, { durable: false });  //声明队列
        await ch.prefetch(1);   //每次最大接收消息数量
        console.log(' [x] Awaiting RPC requests');

        ch.consume(q, function reply(msg) {     //订阅 RPC 队列接收请求
            const n = parseInt(msg.content.toString());
            console.log(" [.] fib(%d)", n);
            const r = fibonacci(n);    //调用本地函数计算结果

            ch.sendToQueue(msg.properties.replyTo,    //将 RPC 请求结果发送到 callback 队列
                new Buffer(r.toString()),
                { correlationId: msg.properties.correlationId }
            );

            ch.ack(msg);
        });
    } catch (error) {
        console.log(error);
    }
})();

function fibonacci(n) {   //时间复杂度比较高
    let cache = {};
    if (n === 0 || n === 1)
        return n;
    else
        return fibonacci(n - 1) + fibonacci(n - 2);
}
```

上述就是一个简单的 RPC 示例。


### 延时队列

某些场景下我们并不希望生产者投递消息后，消费者立即就接收到消息，而是延迟一段时间，比如某个订单提交后十五分钟内未支付则自动取消这种情况就可以用延时队列。

RabbitMQ 本身并没有直接支持延时队列这个功能，我们需要简单的拐个弯间接实现：

![](/images/middleware/rabbitmq4-4.jpeg)

具体流程如上图所示，生产者先将消息投递到一个死信队列中，消息在死信队列中延时，并指定 deadLetterExchange 也就是消息延时结束后重新分发到的交换器，以及 deadLetterRoutingKey，重新分发后的交换器据此将消息分发到另一个队列，消费者订阅此队列以接受消息。

交换器与队列一定是一起出现的，即使我们使用了默认交换器，在代码中无感，也要牢记它的存在。同样上图所示在延时队列中使用的两个交换器都可以为默认交换器，只要我们定义不同的绑定规则即可。

消费者接收消息示例（receive.js）：

```
const amqp = require('amqplib');

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();  //建立信道
        const queueName = 'delay-queue-consumer';
        await ch.assertQueue(queueName, { durable: false });  //声明队列，durable：false 不对队列持久化
        console.log(" [*] Waiting for messages in queue: %s. To exit press CTRL+C", queueName);

        ch.consume(queueName, msg => {   //订阅队列接受消息
            console.log(" [x] Received %s", msg.content.toString());
        }, { noAck: true });   // noAck：true 不进行确认接受应答

    } catch (error) {
        console.log(error);
    }
})()
```

生产者投递消息示例（send.js）：

```
const amqp = require('amqplib');
const EventEmitter = require('events');
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();

(async () => {
    try {
        const conn = await amqp.connect('amqp://localhost');   //建立连接
        const ch = await conn.createChannel();     //建立信道

        const msg = 'Hello world';
        const queueName = 'delay-queue-consumer';
        await ch.assertQueue(queueName, { durable: false });  //消息延时结束后会被转发到此队列，消费者直接订阅此队列即可

        await ch.assertQueue('delay-queue-dead-letter', {   //定义死信队列
            durable: false,
            deadLetterExchange: '',   //直接使用默认交换器
            deadLetterRoutingKey: 'delay-queue-consumer',   //默认交换器路由键就是队列名
            messageTtl: 5000    //延时 ms
        });

        for (let i = 0; i < 5; i++) {
            setTimeout(() => {
                ch.sendToQueue('delay-queue-dead-letter', new Buffer(msg+i));   //发送消息
                console.log(' [x] Sent %s', msg+i);
                if (i == 4) {
                    myEmitter.emit('sent done');
                }
            }, 5000*i)
        }

        myEmitter.on('sent done', async () => {
            await ch.close();   //关闭信道
            await conn.close();  //关闭连接
        });

    } catch (error) {
        console.log(error);
    }
})()
```

示例就写这么多，全文完。