---
title: "RabbitMQ笔记(一)----快速了解"
date: 2022-10-12T14:22:46+08:00
draft: false
tags: ["RabbitMQ", "消息队列","中间件"]
categories: ["RabbitMQ"]
---

## RabbitMQ 是什么

RabbitMQ 是一个消息队列中间件，凭借高可靠，易扩展，高可用和丰富的功能被广泛使用。
上面的描述中有一个概念：消息队列中间件（Message Queue Middleware，简称为MQ），是指利用高效可靠的消息传递机制进行与平台无关的数据交流，并基于数据通信来进行分布式系统的集成。通过提供消息传递和消息排队模型，它可以在分布式环境下扩展进程间的通信。

### 工作模式

消息队列中间件，通常也会被称为消息中间件，一般会有两种工作模式：

- 点对点（p2p）
- 发布/订阅（pub/sub）

### 消息中间件的作用

总结作用如下：

- 解耦
- 冗余
- 扩展性
- 削峰
- 可恢复行
- 顺序保证
- 缓冲
- 异步通信

## RabbitMQ 常用名词概念

RabbitMQ 整体上就是一个生产者与消费者模型，主要负责接收，存储，转发消息，这个过程与邮局非常类似，你将信件送到邮局，邮局暂存并通过邮递员送到收件人手上。

![rabbitmq基本结构](/images/rabbitmq/rabbitmq基本结构.png)

- `Producer`：生产者。
- `Consumer`：消费者。
- `Broker`：消息中间件的服务节点。
- `Connection`： 每个producer（生产者）或者consumer（消费者）要通过RabbitMQ发送与消费消息，首先就要与RabbitMQ建立连接，这个连接就是Connection。Connection是一个TCP长连接。
- `Channel`：是在Connection的基础上建立的虚拟连接，RabbitMQ中大部分的操作都是使用Channel完成的，比如：声明Queue、声明Exchange、发布消息、消费消息等。
- `Virtual Host`: 虚拟主机，一个Broker中可以有多个Virtual host，每个Virtual host都有一套自己的Exchange和Queue，同一个Virtual host中的Exchange和Queue不能重名，不同的Virtual host中的Exchange和Queue名字可以一样。
- `Exchange`：交换器，用于将消息路由到队列中
- `Queue`：队列，一个用来存放消息的队列，生产者发送的消息会被放到Queue中，消费者消费消息时也是从Queue中取走消息。

Exchange  有四种分发消息规则

- `direct`
- `fanout`
- `topic`
- `headers`(用的比较少)

在这里有一个概念非常重要：`Routing key`,当创建好Exchange和Queue之后，需要使用 `Routing key` 将它们绑定起来，producer在向Exchange发送一条消息的时候，必须指定一个Routing key，然后Exchange接收到这条消息之后，会解析`Routing key`，然后根据Exchange和Queue的绑定规则，将消息分发到符合规则的Queue中。

![routing key](/images/rabbitmq/routing_key.png)

### direct

direct的意思是直接的，direct类型的Exchange会将消息转发到指定Routing key的Queue上，Routing key的解析规则为精确匹配。也就是只有当producer发送的消息的Routing key与某个Binding key相等时，消息才会被分发到对应的Queue上。

### fanout

fanout是扇形的意思，该类型通常叫作广播类型。fanout类型的Exchange不处理Routing key，而是会将发送给它的消息路由到所有与它绑定的Queue上。

### topic

topic的意思是主题，topic类型的Exchange会根据通配符对Routing key进行匹配，只要Routing key满足某个通配符的条件，就会被路由到对应的Queue上。通配符的匹配规则如下：

- Routing key必须是一串字符串，每个单词用“.”分隔；
- 符号“#”表示匹配一个或多个单词；
- 符号“*”表示匹配一个单词。

## Hello World

这里使用`Go` 实现一个生产者和消费者，并且Exchange的分发规则使用 `direct`

生产者代码：

```go
package main

import (
	"context"
	"fmt"
	amqp "github.com/rabbitmq/amqp091-go"
	"log"
	"time"
)

func main() {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalln(err)
	}
	defer conn.Close()
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalln(err)
	}
	defer ch.Close()

	// 声明 exchange
	err = ch.ExchangeDeclare(
		"hello",
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)
	if err != nil {
		log.Fatalln(err)
	}
	// 声明队列
	_, err = ch.QueueDeclare(
		"Go",
		false,
		false,
		false,
		false,
		nil,
	)
	if err != nil {
		log.Fatalln(err)
	}
	// 绑定队列
	err = ch.QueueBind(
		"Go",
		"Go",
		"hello",
		false,
		nil,
	)
	if err != nil {
		log.Fatalln(err)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	for i := 0; i < 10; i++ {
		body := fmt.Sprintf("Hello World!-%d!", i)
		err = ch.PublishWithContext(ctx,
			"hello",
			"Go",
			false,
			false,
			amqp.Publishing{
				ContentType: "text/plain",
				Body:        []byte(body),
			},
		)
		if err != nil {
			log.Fatalln(err)
		}
		log.Printf("send message:%s\n", body)
	}

}
```

消费者代码：

```go
package main

import (
	amqp "github.com/rabbitmq/amqp091-go"
	"log"
)

func main() {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalln(err)
	}
	defer conn.Close()
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalln(err)
	}
	defer ch.Close()
	msgs, err := ch.Consume(
		"Go",
		"",
		true,
		false,
		false,
		false,
		nil,
	)
	if err != nil {
		log.Fatalln(err)
	}
	for d := range msgs {
		log.Printf("receive:%s", d.Body)
	}
}

```

如果执行生产者代码之后，你可以在 MQ 的管理页面看到创建了exchange 并创建了队列，队列页面可以看到你发布的消息数量。

![exchange](/images/rabbitmq/hello_exchange.jpg)

![queue](/images/rabbitmq/hello_queue.jpg)
