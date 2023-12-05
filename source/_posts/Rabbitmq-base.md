---
title: RabbitMQ基础
date: 2023-03-09 21:25:46
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/rabbitmq.jpeg
tags:
    - mq
    - rabbitmq
categories:
    - queue
description: RabbitMQ基础
---
# RabbitMQ
## RabbitMQ介绍
RabbitMQ 是采用 Erlang 语言实现 AMQP(Advanced Message Queuing Protocol，高级消息队列协议）的消息中间件，它最初起源于金融系统，用于在分布式系统中存储转发消息。

RabbitMQ 的具体特点可以概括为以下几点：
- 可靠性: 通过自身机制保证其可靠性，如持久化、传输确认及发布确认等
- 灵活的路由: 
- 扩展性: 多个RabbitMQ节点可以组成一个集群
- 高可用性
- 支持多种协议
- 多语言客户端
- 易用的管理界面: 官方提供ui工具
- 插件机制

## RabbitMQ核心概念

RabbitMQ 整体上是一个生产者与消费者模型，主要负责接收、存储和转发消息。机构模型如下图:

![结构模型](/img/rabbitmq.jpg)

### 生产者(Producer) 和 消费者(Consumer)
- 生产者: 消息投放者
- 消费者: 消息消费者

消息一般由两部分组成: **消息头**和**消息体**。消息体不透明。RabbitMQ根据消息头进行转发
### Exchange(交换器)

**RoutingKey**: 生产者将消息交给交换器时，一般会指定一个**RoutingKey**(路由键)，用来指定这个路由规则，而这个路由键需要与绑定键联合使用才能最终生效。

**BindingKey**: 确定了不同的交换器(Exchange)，那么我们还需要将交换器和消息队列进行关联。我们可以通过**BindingKey**(绑定键)进行关联。其中绑定关系是多对多的。多个队列可以使用相同的绑定键(Binding Key)。

交换器是用来接收生产者发送的消息，并将这些消息**路由**给对应的服务器队列，如果路由失败，或许会返回给生产者，或者直接丢弃。也就是说，生产者的消息并不会直接交给Queue(消息队列)，而且需要先通过Exchange(交换器)进行路由。

Exchange分为四个类型，不同的类型，路由规则也不同:
- direct: 把消息路由到绑定键(BindingKey)和路由键(RoutingKey)完全匹配的队列中
- fanout: 把所有的发送到Exchange中的消息路由到它绑定的Queue，不需要其它操作，速度最快。可以用来做广播
- topic: topic类型的交换器在匹配规则上进行了扩展，它与 direct 类型的交换器相似，也是将消息路由到`BindingKey`和`RoutingKey`相匹配的队列中，但这里的匹配规则有些不同，它约定：
  - RoutingKey 为一个点号“．”分隔的字符串（被点号“．”分隔开的每一段独立的字符串称为一个单词），如 “com.rabbitmq.client”、“java.util.concurrent”、“com.hidden.client”
  - BindingKey 和 RoutingKey 一样也是点号“．”分隔的字符串
  - BindingKey 中可以存在两种特殊字符串“*”和“#”，用于做模糊匹配，其中“*”用于匹配一个单词，“#”用于匹配多个单词(可以是零个)
- headers: 不推荐。根据发送的消息内容中的 headers 属性进行匹配。



### 消息队列(Queue)
Queue(消息队列) 用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理，这样避免消息被重复消费。`RabbitMQ`不支持队列层面的广播消费。

### 服务节点(Broker)

对于`RabbitMQ`来说，一个`RabbitMQ Broker`可以简单地看作一个`RabbitMQ`服务节点，或者RabbitMQ服务实例。大多数情况下也可以将一个 RabbitMQ Broker 看作一台`RabbitMQ`服务器

### 死信队列

DLX，全称为`Dead-Letter-Exchange`，死信交换器，死信邮箱。当消息在一个队列中变成死信 (dead message) 之后，它能被重新被发送到另一个交换器中，这个交换器就是`DLX`，绑定`DLX`的队列就称之为死信队列

导致死信的原因:
- 消息被拒绝
- 消息ttl过期
- 队列满了，无法再添加


### 如何保证可靠性

生产者到mq: 事务机制和Confirm机制。这两个是互斥的，不能共存。
mq自身: 持久化，集群，普通模式，镜像模式
mq到消费者: basicAck机制，死信队列，消息补偿机制

![可靠性](/img/rabbitmq-server-client.png)

#### 持久化
如果我们需要将消息进行持久化，一般我们可以在创建队列的进行配置，通过参数`durable`配置实现
```
// 创建的队列的方法，也和下面的注解方法相同
// Params:
//    name – the name of the queue. 
//    durable – 是否持久化，服务器重启，该队列依旧存在 
//    exclusive – 是否独享 
//    autoDelete – 是否自动删除
public Queue(String name, boolean durable, boolean exclusive, boolean autoDelete) {
    this(name, durable, exclusive, autoDelete, null);
}
```
#### confirms or 事务
因为这两个机制是互斥的，我们只需要使用其中一种就可以

**confirms**: 生产者将消息发送给服务器，就会通过回调confirms()来通知生产者
**returnedMessage**: 若交换机不能路由到队列，则回调returnedMessage()来通知生产者

### 几个注解

#### @Queue
    @QueueBinding使用时用来定义队列信息的注解
{% codeblock lang:java %}
public @interface Queue {
    
    // 队列名
    @AliasFor("name")
    String value() default "";
    // 队列名
    @AliasFor("value")
    String name() default "";
    // 指定队列持久化，默认持久化
    String durable() default "";
    // 指定队列排他性，默认不会
    String exclusive() default "";
    // 指定队列如果不用就会被自动删除
    String autoDelete() default "";
    // 是否忽略定义错误
    String ignoreDeclarationExceptions() default "false";
    // 创建时定义的参数
    Argument[] arguments() default {};
    String declare() default "true";
    String[] admins() default {};
    
}
{% endcodeblock %}
#### @Exchange

{% codeblock lang:java %}
public @interface Exchange {
    
    @AliasFor("name")
    String value() default "";
    @AliasFor("value")
    String name() default "";
    // exchange的类型，一个有五种
    String type() default ExchangeTypes.DIRECT;
    // 声明为自动删除则为true
    String autoDelete() default FALSE;
    // 声明为持久化的则为true
    String durable() default TRUE;
    // 声明为内部的则为true
    String internal() default FALSE;
    // 如果声明错误被忽略则为true
    String ignoreDeclarationExceptions() default FALSE;
    // 为true则将exchange被声明为'x-delayed-message'。同时需要broker拥有延迟消息插件
    String delayed() default FALSE;
    Argument[] arguments() default {};
    // 如果admin存在，声明此组件
    String declare() default TRUE;
    // 返回应该声明此组件的管理bean名称列表。默认情况下，所有管理员都会声明它
    String[] admins() default {};

}
{% endcodeblock %}

#### @QueueBinding
    定义一个queue，一个exchange，和一个可选的绑定键
{% codeblock lang:java %}
public @interface QueueBinding {
    Queue value();
    Exchange exchange();
    // 指定绑定的bindingkey或者pattern
    String[] key() default {};
    String ignoreDeclarationExceptions() default "false";
    Argument[] arguments() default {};
    String declare() default "true";
    String[] admins() default {};
}
{% endcodeblock %}

#### @RabbitListener
    标记一个方法做为指定的queues()或者binds()的监听器。都会将其配置放入MethodRabbitListenerEndpoint中进行注册
    当这边注解用于类上时，一个单独的消息监听容器将会为所有带有@RabbitHandler注解的方法提供服务。各个方法都是不能相同的，这样才能为特定入站消息解析
{% codeblock lang:java %}
public @interface RabbitListener {
    // 容器管理中对应这个终端的唯一id
    String id() default "";
    // 负责创建服务消息监听器的容器工厂，如果不指定则启用默认配置
    String containerFactory() default "";
    // 监听器负责的队列。可以是队列名，占位符，表达式
    String[] queues() default {};
    // 如果存在RabbitAdmin在应用中，则可以定义队列在服务上，exchange和queue的routingkey使用queue的名字。与配置bindings() and queues()互斥
    Queue[] queuesToDeclare() default {};
    // 设置为true，只允许单独的消费者使用queues()，也就是并发为1。
    boolean exclusive() default false;
    // 优先级。3.2版本或之上才有用。一般不用修改优先级
    String priority() default "";
    // 引用一个AmqpAdmin。如果监听器正在使用自动删除队列，并且这些队列已配置为条件声明则必须引用。这个admin将声明会在重新声明这些队列，在容器启动或者重启时。
    String admin() default "";
    // 为监听器提供一组QueueBinding。和queues()，queuesToDeclare()冲突
    QueueBinding[] bindings() default {};
    // 如果提供，则此侦听器的侦听器容器将添加到以该值作为其名称的类型为 的 bean 中Collection<MessageListenerContainer>
    String group() default "";
    // 设置为true，使用正常的replyTo或者@SendTo语义通过监听器将异常发送给发送方
    // 设置为false，将异常抛出到监听器容器，并执行正常的重试或者DLQ处理
    String returnExceptions() default "";
    // 设置一个异常处理器。可以是一个简单的string的名字，也可以是一个spel表达式，这个表达式必须能够计算出一个bean的名字或者RabbitListenerErrorHandler的实例
    String errorHandler() default "";
    // 为这个监听器的容器设置并发，默认通过监听器的容器工厂。
    // SimpleMessageListenerContainer: 对于这个监听器容器，可以设置为简单的数字，也可以设置m-n的形式。m标识当前并发数，n标识最大并发数
    // DirectMessageListenerContainer: 对于这个监听器容器，可以设置为consumersPerQueue的属性
    String concurrency() default "";
    // 可以设置为true或者false，来替代容器的默认设置
    String autoStartup() default "";
    // 设置task任务的bean名字为这个监听器容器，覆盖容器工厂上设置的任何执行程序。
    String executor() default "";
    // 覆盖容器中的AcknowledgeMode配置
    String ackMode() default "";
    // 在ReplyPostProcessor发送之前处理响应的bean名称
    String replyPostProcessor() default "";
    // 设置消息转换器
    String messageConverter() default "";
    // 设置回复消息的内容类型。处理不同类型的消息时非常有用
    String replyContentType() default "";
    // 设置为“false”以使用“replyContentType”属性的值覆盖由消息转换器设置的任何内容类型标头。一些转换器，会设置payload的类型和适当的内容头类型。
    String converterWinsContentType() default "true";
}
{% endcodeblock %}

除了这个注解的属性，被这个注解标记的方法，还会给该方法提供灵活的行参，类似于注解`@MessageMapping`

## 多个消息队列优缺点对比整理

### 基础点对比

|    组件    |      kafka      |  RocketMQ   |     RabbitMQ      | ActiveMQ     |
|:--------:|:---------------:|:-----------:|:-----------------:|--------------|
|   推出时间   |      2012       |    2012     |       2007        |              |
|    公司    | linkin开源-apache | 阿里开源-apache | pivotal开源-mozilla |              |
|   开发语言   |  scala & Java   |    Java     |      erlang       |              |
|   吞吐量    |       十万级       |     万级      |        万级         | 单机万级         |
|    延迟    |       毫秒级       |     毫秒级     |        微秒级        |              |
|   主题数    |      几十到几百      |    几百到几千    |     Thousands     |              |
|   高可用    |     高（分布式）      |    高（主从）    |       高（主从）       | 高            |
| **消息丢失** |      理论上不会      |    理论上不会    |         低         | 低            |
| **消息重复** |     理论上有重复      |             |        可控制        |              |
|   推荐度    |       推荐        |     推荐      |        推荐         | 不推荐（性能差，更新慢） |

### 优缺点总结

#### kafka

优点:
- 高吞肚，低延迟
- 高伸缩性
- 高稳定性: 分布式架构，一个数据多个副本
- 持久性，可靠性，可回溯性
- 消息有序: 一个消息被消费且仅有一次
- ui管理节目优秀
- 兼容性好

缺点:
- 不支持消息路由，不支持延迟发送，不支持消息重试；
- 社区更新慢
- 单机分区/队列超过64，load飙高，消息响应需要时间变长

场景选择: 大量数据的互联网服务的数据收集业务 

#### RocketMQ

优点:
- 高吞吐: 单一队列百万消息的堆积能力
- 高伸缩性: 灵活部署
- 高容错性: ack机制，保证消息一定能消费
- 持久化、可回溯: 持久化在磁盘中
- 消息有序: fifo原则和严格的顺序传递
- 提供docker镜像集群部署和功能丰富的dashboard

缺点:
- 不支持消息路由，**支持的客户端语言不多**，主要为Java
- 社区活跃度一般

场景选择: 面向要求可靠性很高的场景，比如金融互联网领域。并且可以参考阿里成熟的实战场景和案例，也可以进行二次开发，定制自己的mq

#### RabbitMQ

优点:
- 支持所有受欢迎的语言
- 支持消息路由: 通过不同交换器支持不同的消息路由
- 消息时序: 可以有延时队列，指定消息的延时时间，过期时间ttl等
- 支持容错: 交付重试已经死信队列
- ui管理节目
- 社区活跃

缺点:
- erlang开发，不利于二次开发
- 吞吐量略低
- 不支持消息有序、持久化不好、不支持消息回溯、伸缩性一般 

场景选型: 并发能力强，性能较好，社区活跃，如果吞吐量不是特别大，没有二次开发需求，可以选择使用
