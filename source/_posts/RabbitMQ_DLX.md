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

DLX其实就是一个普通的交换器，要使用它也很简单，就是在声明某个队列的时候设置其 `deadLetterExchange` 和 `deadLetterRoutingKey` 参数（`deadLetterRoutingKey` 参数可选，表示为 DLX 指定的路由键，如果没有特殊指定，则使用原队列的路由键。）。这样设置后，这个队列的消息一过期，`RabbitMQ` 就会自动地将这个消息重新发布到设置的 DLX 上去，进而被路由到另一个队列，即死信队列。

