---
title: Sentinel + Nacos + Gateway 整合
date: 2023-06-09 17:02:15
tags:
    - Java
    - Sentinel
    - Spring Cloud
    - Nacos
    - Gateway
categories:
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/sentinel.png
---
# SpringCloud Alibaba Sentinel + SpringCloud Alibaba Nacos + SpringCloud Gateway 整合

这套组合，它们兼具了网关，熔断限流，还包括配置中心，注册中心，是作为一个微服务的基础。现在我们就来是用下这套组合，分析下优劣情况。

## 引入jar包
```xml
<dependencies>
    <!-- gateway网关 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!-- nacos注册中心 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <!-- nacos配置中心 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    <!-- actuator健康监控 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- sentinel -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
    <!-- sentinel网关组件 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    </dependency>
    <!-- sentinel使用naocs配置组件 -->
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-datasource-nacos</artifactId>
    </dependency>
</dependencies>
```
## 项目概览
因为是进行流量控制，所有我们就在网关项目上引入上面jar创建项目。并根据官网说明加入如下配置:
{% codeblock lang:java %}
@Configuration
public class GatewayConfiguration {

    private final List<ViewResolver> viewResolvers;
    private final ServerCodecConfigurer serverCodecConfigurer;

    public GatewayConfiguration(ObjectProvider<List<ViewResolver>> viewResolversProvider,
                                ServerCodecConfigurer serverCodecConfigurer) {
        this.viewResolvers = viewResolversProvider.getIfAvailable(Collections::emptyList);
        this.serverCodecConfigurer = serverCodecConfigurer;
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public SentinelGatewayBlockExceptionHandler sentinelGatewayBlockExceptionHandler() {
        // Register the block exception handler for Spring Cloud Gateway.
        return new SentinelGatewayBlockExceptionHandler(viewResolvers, serverCodecConfigurer);
    }
    // 这部分可以不需要，因为我们引入的jar包就帮我们注入了
    @Bean
    @Order(-1)
    public GlobalFilter sentinelGatewayFilter() {
        return new SentinelGatewayFilter();
    }
}
{% endcodeblock %}

## 相关配置

这部分分为三个配置，一个是网关路由的配置，一个sentinel的配置，一个nacos的配置。我们的应用名定义为star-gateway

### 网关配置
```yaml
spring:
  application:
    name: star-gateway
  cloud:
    gateway:
      metrics:
        enabled: true
      globalcors: # 跨域配置
        cors-configurations:
          '[/**]':
            allowedOrigins: "http://localhost"
            allowedMethods:
              - GET
              - POST
              - HEAD
              - OPTIONS
              - PUT
              - PATCH
      enabled: true
      routes: # 路由规则
        - id: demo01 # 路由规则id，同时也是sentinel的资源id
          uri: lb://star-app-demo01
          predicates:
            - Path=/demo01/**
```
### nacos配置
```yaml
spring:
  cloud:
    nacos:
      discovery: # 注册中心
        server-addr: 127.0.0.1:8848
      config: # 配置中心
        server-addr: localhost:8848
        file-extension: yml
```
### sentinel配置
```yaml
spring:
  cloud:
    sentinel:
      filter: # 按照官网说明，引入依赖则置为false
        enabled: false
      datasource: # 多数据与配置，以下为nacos
        local: # 数据源name
          nacos: # 数据源类型，此处为nacos，网关上还有file等类型，具体参考网关。如下配置大部分都与nacos配置相同
            data-id: gateway-sentinel
            server-addr: ${SERVER_ADDR:127.0.0.1:8848}
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: GW_FLOW # 网关流空规则下，不实用flow，使用gw-flow
      transport: # dashboard服务地址。对应的dashboard启动配置为-Dserver.port=8080
        dashboard: 127.0.0.1:8080
```
### nacos中配置sentinel规则
启动nacos，进入nacos控制台，在配置管理模块新增配置。
![sentinel-nacos](/img/sentinel-nacos-before.png)

- data-id: gateway-sentinel。和sentinel配置中nacos的配置相同
- group：DEFAULT_GROUP。和sentinel配置中nacos的配置相同
- data-type: json。选择json
- 配置内容: 可以配置多条规则。如下：
```json
[
  {
    "resource": "demo01",
    "grade": 1,
    "count": 3,
    "strategy": 0,
    "controlBehavior": 0
  }
]
```
**注:** resource资源名，对应路由名
  ![sentinel-nacos](/img/sentinel-nacos-after.png)

### sentinel-dashboard
dashboard的具体使用就不赘述了，我们可以通过如下命令启动
```shell
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.6.jar
```
|配置|                  说明                  |
|:--:|:------------------------------------:|
| -Dserver.port  | dashboard服务端口。Java服务中dashboard配置的端口号 | 
| -Dcsp.sentinel.dashboard.server |              控制台的地址和端口               |

启动之后界面如下: 我们可以看到sentinel自己的运行情况和配置相关的规则
![sentinel-dashboard.png](/img/sentinel-dashboard.png)

## 启动应用
配置完如上配置，我们就可以启动网关等应用了。因为要模拟请求，所以我多创建了一个test应用，用来进行网关转发请求。这时候你如果观察dashboard控制台，你会发现只有一个项目`sentinel-dashboard`，很疑惑，我们的`star-gateway`项目为什么没有，经过我查找资料和身体力行，我发现原来在sentinel的dashboard中，并不是应用启动就会注册，只有当第一个请求进来，才会出现我们注册的应用。如下:
![sentinel-dashboard-register](/img/sentinel-dashboard-register.png)

我们可以发现我们的应用也出现了，nacos中的配置也自动注册到了对应的规则中，如果我们要改变配置，可以在dashboard中修改，也可以nacos中修改。但是，sentinel-dashboard上修改不能同步到nacos，反之则可以，**所以推荐在nacos上修改**

dashboard配置规则如下:
![sentinel-config](/img/sentinel-config.png)

## 应用请求

我们通过多次尝试请求url`http://localhost:8081/demo01/index` ，根据配置，通过`QPS`来做限流，阀值为3，超出阀值之后就会出现如下报错，这样证明我们的限流配置生效了。
![sentinel-block](/img/sentinel-block.png)