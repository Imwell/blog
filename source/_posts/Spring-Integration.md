---
title: Spring Integration
date: 2021-01-25 16:49:53
tags: 
    - Java
    - Spring
categories:    
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# Spring Integration
## 简介
Spring Integration就类似一个水电系统，各层楼的控制，分流，聚合，过滤，沉淀，消毒，排污这里的每一个环节都类似一个系统服务，可能是jms,可能是redis,可能是MongoDB，可能是Tcp/UDP，可能是jo，可能是我们系统服务的任何一个模块。
那么Spring Integration扮演的角色就是将这些功能能够连接起来组成一个完整的服务系统，实现企业系统的集成的解决方案。
就像管道一样将各个模块连接到一起，管道能够连接到千家万户需要很多零件水表，分头管，水龙头，管道开关等等这些就是Spring Integration的主要组件
## 功能介绍
Spring Integration主要包括：
- Message：消息
- Channel：消息传输的通道
- Channel Adapter  消息入口和出口的，连接到某些外部系统或传输方式
### 详细功能
1. 通道(Channel)：消息传输的通道，消息传入通道，然后再从通道里面拿取消息。Spring Integration提供了多种通道的实现，默认使用的是DirectChannel。
    1. PublishSubscribeChannel：订阅发布通道。通道中的消息会被发送到一个或多个消费者中。 
    2. QueueChannel：队列通道。消息会先进入一个队列，然后按照先进先出的规则，将消息发送到其中一个消费者上
    3. DirectChannel：和PublishSubscribeChannel，只发送到一个消费者上，和发送者同线程的消费者
    4. FluxMessageChannel：反应式流的发布者通道，基于Reactor实现的    
2. 通道适配器(Channel Adapter)：连接外部系统和写出到外部系统，分为入站（inBound）和出站（outBound）。
是配置器是单向的，入站通道适配器只支持接收消息，出站通道适配器只支持输出消息。支持的适配类型如下：
{% blockquote %}
AMQP、Spring应用事件、文件系统(File)、FTP/FTPS、GemFire、HTTP、JDBC、JPA、JMS、Mail、MongoDB、Redis、RMI、SFT、STOMP、Stream、Syslog、TCP/UD、Twitter、XMPP、WebServices(SOAP、REST)、WebSocket、Zookeeper、WebFlux
{% endblockquote %}
3. 网关（Gateway）：将数据传入集成流中。提供了双向的网关，请求和响应可以一起存在
4. 路由（Router）：将消息路由至一个或多个通道，根据消息的载荷或者头信息进行路由
5. 过滤器（Filter）：通过一些断言，对消息进行过滤
6. 切分器（Splitter）：将一个消息切分为两个或者更多的消息单独处理，然后发送给不同的通道
7. 聚合器（Aggregator）：切分器的反向操作，接受一个集合类型的参数，返回一个消息
8. 转换器（Transformer）：将消息的载荷从一个类型转换为另一种类型
9. 服务激活器（Service Activator）：可以通过一个Spring Bean对消息进行处理，然后输出到指定的通道上。就是将输入和输出连接好，这样就可以运作了。

### 流程梳理
1. 通过网关：网关 -> 路由 -> 过滤器 -> 转换器 -> 服务激活
2. 通过适配器：适配器 -> 路由 -> 过滤器 -> 转换器 -> 服务激活

## 实例
### 简单查询数据库
- **网关**（入口）
{% codeblock lang:java %}
@Component
@MessagingGateway(defaultRequestChannel = "routerIn") // 标识消息将通过方法被发送去的默认通道
public interface OrderGateway {
    // 定义接口
    public Order get(Long id);

}
{% endcodeblock %}
- **配置文件**
{% codeblock lang:java %}
@Configuration
public class TacoOrderIntegrationConfig {
  // 放入路由，过滤器等配置
  
  // 定义消息通道的实现，作为通道
  @Bean
  public MessageChannel filterIn() {
    return new DirectChannel();
  }

  @Bean
  public MessageChannel routerIn() {
    return new DirectChannel();
  }

  @Bean
  public MessageChannel handlerIn() {
    return new DirectChannel();
  }
}
{% endcodeblock %}
- **路由**
{% codeblock lang:java %}
    /**
    * @Router注解这只能表示是从哪个通道里面获取信息，并不能代表这个通道，
    * 需要单独创建对应通道的Bean对象才行，下面的相关配置也类似
    */
    @Bean
    @Router(inputChannel = "routerIn")
    public AbstractMessageRouter orderRouter() {
        return new AbstractMessageRouter() {
        @Override
        protected Collection<MessageChannel> determineTargetChannels(Message<?> message) {
            Long i = (Long) message.getPayload();
            if (i < 2) { // 简单处理
                return Collections.singleton(filterIn()); // 定义接下来要发送去的通道
            }
            return null;
        }
    };
{% endcodeblock %}
- **过滤器**
{% codeblock lang:java %}
@Filter(inputChannel = "filterIn", outputChannel = "handlerIn") // 标记进入和输出的通道
public boolean orderFilter(Long id) { // 简单逻辑处理
    return id <= 10;
}
{% endcodeblock %}
- **服务**
{% codeblock lang:java %}
@Bean
@ServiceActivator(inputChannel = "handlerIn")
public GenericHandler<Long> get(OrderRepository orderRepository) {
    return ((payload, headers) -> orderRepository.findById(payload).orElse(null)); // 放入载荷，并返回，表示处理流的终点
}
{% endcodeblock %}
### 购买饮料
