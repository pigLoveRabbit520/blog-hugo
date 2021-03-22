title: RabbitMQ四五事之死信队列
author: Salamander
tags:
  - RabbitMQ
categories:
  - RabbitMQ
date: 2021-03-20 20:27:00
---
<img src="/images/RabbitMQ-Logo.png" width="800px">  


## 业务需求
有时候我们需要某些任务定时执行，譬如取消订单，5分钟没支付，这个订单就被取消。简单实现的话，我们可以使用`Redis`或Linux的**crontab**来实现，而对于RabbitMQ，我们则可以用它的`死信队列`来实现定时任务。

<!-- more -->


## DLX
RabbitMQ 中有一种交换器叫 DLX，全称为 `Dead-Letter-Exchange`，可以称之为死信交换器。当消息在一个队列中变成死信（dead message）之后，它会被重新发送到另外一个交换器中，这个交换器就是 DLX，绑定在 DLX 上的队列就称之为死信队列。  
消息变成死信一般是以下几种情况：

* 消息被拒绝，并且设置 requeue 参数为 false
* 消息过期
* 队列达到最大长度

DLX其实就是一个普通的交换器，要使用它也很简单，就是在声明某个队列的时候设置其 `deadLetterExchange` 和 `deadLetterRoutingKey` 参数，`deadLetterRoutingKey` 参数可选，表示为 DLX 指定的路由键，如果没有特殊指定，则使用原队列的路由键。这样设置后，这个队列的消息一过期，`RabbitMQ` 就会自动地将这个消息重新发布到设置的 DLX 上去，进而被路由到另一个队列，即死信队列。  

## 简单例子
用之前文章[RabbitMQ二三事](https://segmentfault.com/a/1190000018685360)快速启动RabbitMQ的服务，再把[RabbitMQ三四事](https://segmentfault.com/a/1190000019227064)的代码改造下。  
**producer.js**  
```
const config = require("./config");
const amqp = require('amqplib');

async function getMQConnection() {
    return await amqp.connect({
        protocol: 'amqp',
        hostname: config.host,
        port: config.port,
        username: config.user,
        password: config.pass,
        locale: 'en_US',
        frameMax: 0,
        heartbeat: 5, // 心跳
        vhost: config.vhost,
    })
}

async function run(rmqConn, msgObj) {
    const noramlQueue = 'noramlQu';
    const noramlExchange = 'noramlEx';
    const exchangeDLX = 'testExDLX';
    const queueDLX = 'testQueueDLX';
    const routingKeyDLX = 'testRoutingKeyDLX';

    try {
        const channel = await rmqConn.createChannel();

        // 声明死信交换器和队列，就跟普通的一样
        await channel.assertExchange(exchangeDLX, 'direct', { durable: true, autoDelete: false });
        await channel.assertQueue(queueDLX, {durable: true, autoDelete: false, });
        await channel.bindQueue(queueDLX, exchangeDLX, routingKeyDLX);


        // 普通交换器和队列
        await channel.assertExchange(noramlExchange, 'direct', { durable: true, autoDelete: false })
        await channel.assertQueue(noramlQueue, {durable: true, autoDelete: false,
            deadLetterExchange: exchangeDLX,
            deadLetterRoutingKey: routingKeyDLX,
        });  // 队列设置DLX
        await channel.bindQueue(noramlQueue, noramlExchange, noramlQueue);

        // 发送消息
        await channel.publish(noramlExchange, noramlQueue, Buffer.from(msgObj.content), {
            expiration: msgObj.expiration, // 过期时间，ms
            persistent: true, 
            mandatory: true,
        });
        console.log('send message successfully.')
        await channel.close();
    } catch(err) {
        console.log('send message failed:' + err.message)
    }
}

async function testSend() {
    const conn = await getMQConnection()
    await run(conn, {
        content: (new Date()).toLocaleString(),
        expiration: '3000',
    })
    await conn.close()
}
testSend();
```
**consumer.js**  
```
const config = require("./config");
const amqp = require('amqplib');

async function getMQConnection() {
    return await amqp.connect({
        protocol: 'amqp',
        hostname: config.host,
        port: config.port,
        username: config.user,
        password: config.pass,
        locale: 'en_US',
        frameMax: 0,
        heartbeat: 5, // 心跳
        vhost: config.vhost,
    })
}

async function run(rmqConn) {
    const noramlQueue = 'noramlQu';
    const noramlExchange = 'noramlEx';
    const exchangeDLX = 'testExDLX';
    const queueDLX = 'testQueueDLX';
    const routingKeyDLX = 'testRoutingKeyDLX';

    try {
        const channel = await rmqConn.createChannel();

        // 声明死信交换器和队列，就跟普通的一样
        await channel.assertExchange(exchangeDLX, 'direct', { durable: true, autoDelete: false });
        await channel.assertQueue(queueDLX, {durable: true, autoDelete: false, });
        await channel.bindQueue(queueDLX, exchangeDLX, routingKeyDLX);

        // 处理死信队列消息
        await channel.consume(queueDLX, msg => {
            console.log(`[${(new Date()).toLocaleString()}] consumer msg：`, msg.content.toString());
        }, { noAck: true });
    } catch(err) {
        console.log('consume message failed:' + err.message)
    }
}

async function testConsume() {
    const conn = await getMQConnection();
    console.log('begin consuming messages...');
    await run(conn);

    process.on('SIGINT', () => {
        console.log('stop consumer.')
        conn.close();
    });
}
testConsume();
```
**config.js**  
```
module.exports = {
    host: '127.0.0.1',
    port: 5672,
    user: 'test',
    pass: '************',
    vhost: '/'
}
```

上面的代码，我们让消息3s后过期，先启动**消费者**，再启动生产者，我们可以看到消息3s后过期:   


```
$ node consumer.js 
begin consuming messages...
[2021/3/20 下午3:22:29] consumer msg： 2021/3/20 下午3:22:26
```

## 死信队列问题
RabbitMQ中，每个消息的过期不是独立的，一个队列里的某个消息即使比同队列中的其它消息提前过期，也不会优先进入到死信队列，**只有当过期的消息到了队列的顶端，才会被真正的丢弃或者进入死信队列**。  
我们把**生产者**的代码调整下，先发一个20s过期的消息，再发一个3s过期的消息
```
....
async function testSend() {
    const conn = await getMQConnection()
    await run(conn, {
        content: (new Date()).toLocaleString() + ' 20s过期 ',
        expiration: '20000',
    })
    await run(conn, {
        content: (new Date()).toLocaleString() + ' 3s过期 ',
        expiration: '3000',
    })
    await conn.close()
}
```

观察**消费者**输出： 

```
$ node consumer.js 
begin consuming messages...
[2021/3/21 下午6:10:21] consumer msg： 2021/3/21 下午6:10:01 20s过期 
[2021/3/21 下午6:10:21] consumer msg： 2021/3/21 下午6:10:01 3s过期
```
可以发现，3s过期的消息并没有先被消费，而是只能前面的20s过期的消息先过期，它才会被检查是否过期。  
究其本质的话，RabbitMQ 的队列是一个 `FIFO` 的有序队列，投入的消息都顺序的压进 MQ 中。而 RabbitMQ 也只会对队列顶端的消息进行超时判定，所以就出现了上述的情况。  
所以对于固定时间的延时任务的话，例如下单后半小时未支付则关闭订单这种场景，RabbitMQ无疑可以很好的承担起这个需求，但对于需要每条消息的死亡相互独立这种场景，RabbitMQ还是无法满足的。

## 解决队列消息非异步

RabbitMQ 本身没有这种功能，但是它有个插件可以解决这个问题：**rabbitmq_delayed_message_exchange**，[地址](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange) 。  
这个插件的介绍如下：
> A user can declare an exchange with the type x-delayed-message and then publish messages with the custom header x-delay expressing in milliseconds a delay time for the message. The message will be delivered to the respective queues after x-delay milliseconds.  

这个插件增加了一种新类型的`exchange`：**x-delayed-message**，然后只要发送消息时指定的是这个交换机，那么只需要在消息 header 中指定参数 x-delay [:毫秒值] 就能够实现每条消息的异步延时。  

### 添加插件
用了Docker之后，添加这个插件非常简单，添加`Dockerfile`：
```
FROM rabbitmq:3.8.12-management
COPY ./rabbitmq_delayed_message_exchange-3.8.9-0199d11c.ez /plugins
RUN rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```
插件可以在[Release页](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases)下载。  
**docker-compose.yml**改下：
```
version: "2"
services:
  mq:
    build: .
    restart: always
    mem_limit: 2g
    hostname: mq1
    volumes:
      - ./mnesia:/var/lib/rabbitmq/mnesia
      - ./log:/var/log/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
    ports:
      - "55672:15672"
      - "5672:5672"
    environment:
      - CONTAINER_NAME=rabbitMQ
      - RABBITMQ_ERLANG_COOKIE=3t182q3wtj1p9z0kd3tb
```
这样插件就安装成功了。  

### 修改代码
**producer.js**
```
const config = require("./config");
const amqp = require('amqplib');

async function getMQConnection() {
    return await amqp.connect({
        protocol: 'amqp',
        hostname: config.host,
        port: config.port,
        username: config.user,
        password: config.pass,
        locale: 'en_US',
        frameMax: 0,
        heartbeat: 5, // 心跳
        vhost: config.vhost,
    })
}

async function run(rmqConn, msgObj) {
    const exchangeDelay = 'testExNewDelay';
    const queueDLX = 'testQueueDLX';
    const routingKeyDLX = 'testRoutingKeyDLX';

    try {
        const channel = await rmqConn.createChannel();

        // x-delayed-message类型的exchange
        await channel.assertExchange(exchangeDelay, 'x-delayed-message', { durable: true, autoDelete: false, arguments: {'x-delayed-type':  "direct"} });
        await channel.assertQueue(queueDLX, {durable: true, autoDelete: false, });
        await channel.bindQueue(queueDLX, exchangeDelay, routingKeyDLX);

        // 发送消息
        await channel.publish(exchangeDelay, routingKeyDLX, Buffer.from(msgObj.content), {
            headers: {"x-delay": msgObj.expiration}, // ms
            persistent: true,
            mandatory: true,
        });
        console.log('send message successfully.')
        await channel.close();
    } catch(err) {
        console.log('send message failed:' + err.message)
    }
}

async function testSend() {
    const conn = await getMQConnection()
    await run(conn, {
        content: (new Date()).toLocaleString() + ' 20s过期 ',
        expiration: '20000',
    })
    await run(conn, {
        content: (new Date()).toLocaleString() + ' 3s过期 ',
        expiration: '3000',
    })
    await conn.close()
}
testSend();
```

`x-delayed-type`告诉插件在给定的延迟时间过去之后，exchange应该跟`direct`，`fanout`，`topic`中的exchange路由功能一样。  

**consumer.js**  
```
const config = require("./config");
const amqp = require('amqplib');

async function getMQConnection() {
    return await amqp.connect({
        protocol: 'amqp',
        hostname: config.host,
        port: config.port,
        username: config.user,
        password: config.pass,
        locale: 'en_US',
        frameMax: 0,
        heartbeat: 5, // 心跳
        vhost: config.vhost,
    })
}

async function run(rmqConn) {
    const exchangeDelay = 'testExNewDelay';
    const queueDLX = 'testQueueDLX';
    const routingKeyDLX = 'testRoutingKeyDLX';

    try {
        const channel = await rmqConn.createChannel();

        // x-delayed-message类型的exchange
        await channel.assertExchange(exchangeDelay, 'x-delayed-message', { durable: true, autoDelete: false, arguments: {'x-delayed-type':  "direct"} });
        await channel.assertQueue(queueDLX, {durable: true, autoDelete: false, });
        await channel.bindQueue(queueDLX, exchangeDelay, routingKeyDLX);

        // 处理死信队列消息
        await channel.consume(queueDLX, msg => {
            console.log(`[${(new Date()).toLocaleString()}] consumer msg：`, msg.content.toString());
        }, { noAck: true });
    } catch(err) {
        console.log('consume message failed:' + err.message)
    }
}

async function testConsume() {
    const conn = await getMQConnection();
    console.log('begin consuming messages...');
    await run(conn);

    process.on('SIGINT', () => {
        console.log('stop consumer.')
        conn.close();
    });
}

testConsume();
```

### 测试效果
执行**生产者**代码之后，我们可以看到**消费者**输出：
```
$ node consumer.js 
begin consuming messages...
[2021/3/22 下午2:16:37] consumer msg： 2021/3/22 下午2:16:34 3s过期 
[2021/3/22 下午2:16:54] consumer msg： 2021/3/22 下午2:16:34 20s过期
```
可以发现，消息已经独立的过期了。

### 局限性
没有什么东西是完美的，这个插件也不例外。看下这个插件的`Performance Impact`部分：
```
For each message that crosses an "x-delayed-message" exchange, 
the plugin will try to determine if the message has to be expired by making sure the delay is within range, 
ie: Delay > 0, Delay =< ?ERL_MAX_T (In Erlang a timer can be set up to (2^32)-1 milliseconds in the future).
```
延迟时间最大为 (2^32)-1 毫秒，大约 49 天。另外这个插件也不适合大量延迟消息（例如数十万或数百万）的场景，`Limitations`也写了：
```
Current design of this plugin doesn't really fit scenarios with a high number of delayed messages 
(e.g. 100s of thousands or millions). See #72 for details.
```

参考：
* [RabbitMQ 死信机制真的可以作为延时任务这个场景的解决方案吗？](https://www.skypyb.com/2020/01/jishu/1206/)
* [RabbitMQ 延迟队列插件 x-delay Bug](http://blog.lbanyan.com/rabbitmq_delay/)