---
title: SpringCloud系列-熔断器
date: 2022-06-03 16:51:05
tags:
    - SpringCloud
    - Java
categories:
    - SpringCloud
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# 熔断器

## 什么是熔断器？
熔断器，如果电箱中的保险丝一般，如果出现预料之外的情况，可以通过切断当前电路，进而保护当前工作人员或者环境，在软件中，熔断器也是起相同作用。如果出现API调用无限重试的情况，熔断措施就会生效，停止重试，继续后续调用

## 断路器模式

### 熔断器的状态: 微软的云计算设计模式对熔断器的实现
1. Closed: 默认模式。维护一个失败阀值计数器，如果超出这个阀值，就会修改状态，进行open状态
2. Open: 操作不会执行，而是立即失败。维护一个计时器，当时间达到一定阀值，就会从open进行half-open状态
3. Half-Open: 只要操作一次就进入open状态，否则增加操作成功的次数，当次数达到一定阀值，进入Closed状态并重置失败次数

一般情况下，熔断器就是这样工作的，当大量请求进入，短时间出现问题，就会触发熔断，走入我们设定好的逻辑，依次来避免服务器崩溃的情况。

## Sentinel
    面向分布式服务器架构的轻量级高可用流量控制组件
### Sentinel的核心功能
1. 流量控制: 根据资源的配置对流量进行控制
2. 熔断降级: 调用链路中某个资源不稳定，对其进行控制，并让其快速失败，避免影响到其它资源
3. 系统负载保护: 提供系统纬度的自适应保护能力

进一步总结:
- 资源: 熔断针对的就是资源
- 规则: 针对资源对应的策略
- 数据源: 规则的数据都从数据源加载

## Alibaba Sentinel
    是Sentinel和Spring Cloud体系的整合，该组件整合了Spring MVC、Netflex Zuul、Spring Cloud Gateway、Spring Cloud Circuit Breaker、RestTemplate、OpenFeign、WebFlux等组件

### 熔断

**核心类**: 降级规则
{% codeblock lang:java %}
// 当资源处于不稳定状态下就会使用降级，这些资源将在下一个时间段内进行降级。有如下两种方法可以断定是否处在不稳定状态:
// 1. 平均响应时间(响应时间简称rt): 当平均rt超过阀值时，资源进入准降级状态，如果接下来的5个请求的rt仍然超过这个阀值，该资源被降级，那么意味着下个时间段内，对该资源的访问都会被拒绝
// 2. 异常比例: 每秒异常次数与成功次数qps的比例超过阀值，那么下个时间段将阻塞对该资源的访问
public class DegradeRule extends AbstractRule {
    
    // Circuit breaking策略: 0 平均时间；1 异常比例；2 异常数 
    private int grade = RuleConstant.DEGRADE_GRADE_RT;
    // 阀值。不同策略对应的阀值意义不同
    private double count;
    // 断路器开启时，恢复超时，以秒为单位，超过这个时间，将会转为half-open状态，尝试请求
    private int timeWindow;
    // 触发断路器的最小请求数(在活动统计时间跨度内)
    private int minRequestAmount = RuleConstant.DEGRADE_DEFAULT_MIN_REQUEST_AMOUNT;
    // rt模式下慢请求的比例
    private double slowRatioThreshold = 1.0d;
    // 以毫秒为单位统计时间间隔
    private int statIntervalMs = 1000;
    
    // 实现了父类的equals方法
    @Override
    public boolean equals(Object o) {
        if (this == o) { return true; }
        if (o == null || getClass() != o.getClass()) { return false; }
        if (!super.equals(o)) { return false; }
        DegradeRule rule = (DegradeRule)o;
        return Double.compare(rule.count, count) == 0 &&
            timeWindow == rule.timeWindow &&
            grade == rule.grade &&
            minRequestAmount == rule.minRequestAmount &&
            Double.compare(rule.slowRatioThreshold, slowRatioThreshold) == 0 &&
            statIntervalMs == rule.statIntervalMs;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(super.hashCode(), count, timeWindow, grade, minRequestAmount,
            slowRatioThreshold, statIntervalMs);
    }
}
{% endcodeblock %}

#### 整合OpenFeign与RestTemplate

除了我们显示的调用，还支持使用OpenFeign与RestTemplate，这样就能更方便我们服务调用的安全。

##### OpenFeign
OpenFeign是基于注解使用，我们在融合sentinel时，也只需要通过配置注解上的`fallbackFactory`属性，或者`fallback`注解，都能完成降级之后的调用配置。

注意:
1. fallbackFactory: 是需要通过实现FallbackFactory接口，通过实现`create()`方法使用，可以通过判断抛出异常类型来定义具体的返回
2. fallback: 只需要配置注解类的默认实现类就行

##### RestTemplate
与OpenFeign大致类似。但是默认请求code中，除了5xx，4xx也会抛出异常，通过如下设置，就可以配置自己需要的错误码

{% codeblock lang:java %}
@Configuration
public class SentinelConfig{
    
    @Bean
    @SentinelRestTemplate
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        template.setErrorHandler(new ResponseErrorHandler() {
            @Override
            public boolean hasError(ClientHttpResponse response) throws IOException {
                return response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR;
            }
            @Override
            public void handleError(ClientHttpResponse response) throws IOException {
                throw new IllegalStateException("illegal status code");
            }
        });
        return template;
    }
}
{% endcodeblock %}


### 限流
除了熔断功能，sentinel还提供了限流功能，同时，它也和Netflix Zuul、Spring Cloud Gateway都进行了整合。官方和提供了一个轻量级的开源Dashboard控制台，以方便配置相关限流策略。

限流支持的纬度包含:
- qps设置
- 统计时间窗口设置，默认1s
- 来源ip限流
- host限流
- 任意header限流
- 任意cookie限流
- 任意url query parameter限流

