---
title: Spring WebFlux
date: 2021-01-26 23:07:06
tags:
    - Java
    - Spring
categories:    
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# WebFlux简介
WebFlux是Spring5之后支持的新特性。能够替代传统的SpringMVC，作为新的Web架构。它和SpringMVC非常相似，但是核心又是不同的。传统的Web框架都是阻塞、多线程，一个请求一个线程。
WebFlux是Spring5引入的一个非阻塞、异步的Web框架，它是基于Reactor，能够解决Web应用和Api中对更好的扩展性的要求。
WebFlux默认使用Netty作为Web服务器，Netty是一个基于NIO的框架，很适合与WebFlux一起使用。WebFlux并不是和Netty耦合在一起的，它还能选择Tomcat、Undertow、Jetty等Web服务器。
WebFlux与SpringMVC系出同门，很多的核心组件都是共用的，同时也不会绑定ServletAPI，它构建在Reactive HTTP API之上，它们的功能是一直的，就是采用的是反应式的方法
# WebFlux基础执行流程
- 处理核心类

SpringMVC|WebFlux 
:--:|:--:
DispatcherServlet|DispatcherHandler
  
因为它与SpringMVC系出同门，那么在执行流程上也是有类似的地方，首先，从核心的类上来看，SpringMVC的对请求处理的核心类是`DispatcherServlet`，WebFlux的是`DispatcherHandler`，从命名上来，它们就是属于用一个规范的。
DispatcherHandler是作为HTTP的中央调度器的存在。它会发送给已注册的处理程序处理请求，从而提供方便的映射工具。其中包含了三个重要处理请求的组件：
- HandlerMapping -- 映射处理器，映射到对应的处理方法
- HandlerAdapter -- 正真负责处理请求
- HandlerResultHandler -- 响应结果处理器

接下来，重点看看WebFlux的执行流程。
{% codeblock lang:java %}
/**
* 处理HTTP请求的处理的基础类
*/
public interface WebHandler {

	/**
	 * Handle the web server exchange.
	 * @param exchange the current server exchange
	 * @return {@code Mono<Void>} to indicate when request handling is complete
	 */
	Mono<Void> handle(ServerWebExchange exchange);

}
{% endcodeblock %}

{% codeblock lang:java %}
/**
* DispatcherHandler实现类对处理HTTP请求的具体实现，也是处理HTTP请求的核心方法
* @param exchange 包装HTTP请求和响应的对象
* @return {@code Mono<Void>}
*/
public Mono<Void> handle(ServerWebExchange exchange) {
    if (this.handlerMappings == null) { // 1. 判断映射处理器是否存在
        return createNotFoundError();
    }
    return Flux.fromIterable(this.handlerMappings) // 2. 通过集合构建Flux流
        .concatMap(mapping -> mapping.getHandler(exchange)) // 3. 通过请求获取对应的映射处理器，严格保证顺序是因为在一个系统中可能存在一个Url有多个能够处理的HandlerMapping的情况
        .next() // 4. 通过next()获取下一个
        .switchIfEmpty(createNotFoundError()) // 5. 如果不到值则抛出错误
        .flatMap(handler -> invokeHandler(exchange, handler)) // 6. 真正去执行对应请求
        .flatMap(result -> handleResult(exchange, result)); // 7. 响应结果
}
{% endcodeblock %}
  
**总结**：
1. 请求进入，通过请求获取对应的HandlerAdapter
2. 通过HandlerAdapter，触发正真的请求处理，得到HandlerResult
3. 通过HandlerResultHandler，找到HandlerResult对应的响应结果处理器去处理，如果有错则抛出，没有错就讲数据写入response中返回

注：concatMap与flatMap基础是功能是一致的，但是前者有序，所有在获取对应映射时使用
