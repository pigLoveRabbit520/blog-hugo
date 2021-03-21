title: RabbitMQ四五事之死信队列
author: Salamander
tags:
  - RabbitMQ
categories: []
date: 2021-03-20 23:27:00
---
## 业务需求
有时候我们需要某些任务定时执行，譬如取消订单，5分钟没支付，这个订单就被取消。简单实现的话，我们可以使用`Redis`或Linux的**crontab**来实现，而对于RabbitMQ，我们则可以用它的`死信队列`来实现定时任务。


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

async function testSend() {
    const conn = await getMQConnection();
    console.log('begin consuming messages...');
    await run(conn);

    process.on('SIGINT', () => {
        console.log('stop consumer.')
        conn.close();
    });
}
testSend();
```
**config.js**  
```
module.exports = {
    host: '127.0.0.1',
    port: 56720,
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


