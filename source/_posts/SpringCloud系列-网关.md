---
title: SpringCloud系列-网关
date: 2021-02-18 15:52:08
tags:
    - Java
    - SpringCloud
    - Gateway
categories:
    - SpringCloud
    - Gateway
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# 网关
在现在微服务的背景下，一个系统会有不止一个服务存在，但是像安全认证，流量控制，日志，监控这些功能是每个服务都需要的，没有一个统一的网关，我们就需要在每个系统都接入一个这样的实现，这使得我们的维护成本变得很高，所以，就出现了网关，帮我们进行统一管理

一般一个网关的功能包含:
- 请求转发
- 负载均衡: 根据每个服务的具体情况配置负载均衡策略
- 安全认证: 对用户请求进行安全校验
- 参数校验
- 日志记录
- 监控告警
- 流量控制
- 熔断降级: 实时监控请求的统计信息，达到配置的失败阀值后，自动熔断，返回默认值
- 响应缓存
- 响应聚合
- 灰度发布: 通过导流到不同版本实现基础的灰度发布
- 异常处理
- 协议转换

常见的网关:
- netflix zuul
- SpringCloud Gateway
- APISIX
- KONG

## Gateway

核心:
- Route(路由): 网关的基本模块。通过一个id或是目标uri或是Predicate的集合或者filter集合定义
- Filter(过滤器): 用特定工厂构建的GatewayFilter的实例
- Predicate(断言): 运行匹配http请求中的任何内容，例如headers和params

### 引入

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

// 如果是通过loadbalancer的方法进行路由，需要引入下面jar，不然lb失效
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```
