---
title: Sentinel (一)
date: 2023-05-04 14:05:17
tags:  
    - Java
    - Sentinel
    - Spring Cloud
categories:
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/sentinel.png
---
# Sentinel

sentinel是**面向分布式、多语言异构化服务架构的流量治理组件**，主要以流量为切入点，帮助开发者保障微服务的稳定性。它包括如下几个纬度:
- 流量路由
- 流量控制
- 流量整形
- 熔断降级
- 系统自适应过载保护
- 热点流量防护

## Sentinel的基础概念

- 资源: 代表被sentinel保护的对象，可以是Java程序中的任意内容。大部分情况下可以uri，签名等
- 规则: 对资源进行条件设定，可以动态调整的规则

### 核心功能

#### 流量控制
根据系统的处理能力对流量进行控制。控制面有如下几个角度:
- 资源的调用关系: 调用链路，系统间调用 
- 运行指标: qps，线程池，系统负载等
- 控制的效果: 限流，冷启动，排队

#### 熔断降级
如果某个资源发生异常，并堆积了很多请求，那么可能会影响到其它系统的稳定性。如果异常比例升高，则对这个资源进行限制，并让其快速请求失败

设计的理念:
- 并发线程数进行限制: 请求堆积，线程数就会越来越多，对其进行限制，超出上限之后，对其它的请求进行拒绝
- 通过响应时间对资源进行降级: 当依赖的资源出现响应时间变长，所有对该资源的访问就会拒绝

sentinel于hystrix对比: 
- hystrix使用的是线程间隔离，但是线程间切换的成本会增加，而且需要对其预先分配内存空间
- sentinel的两种设计方式，避免以上问题

#### 系统负载保护
提供系统纬度的保护措施。当系统的流量已经处于它的承受范围之外，网络负载均衡就会进行流量转发，如果多个转发服务都处于流量颠覆，那么就会导致整个集群不可用。sentinel就会提供相应的保护机制，使流量处于一种平衡状态

### Sentinel
在sentinel中，都是以资源主体，分为一个个的资源或者entry。entry是由框架创建或者API创建。整个工作流是通过多个slot来完成的，主要有:
- NodeSelectorSlot: 收集资源路径。每个资源都是生成一个树状结构然后存储起来，用于根据调用进行限流降级
- ClusterBuilderSlot: 存储资源的统计信息以及调用者信息。
- StatisticsSlot: 记录，统计。核心之一，统计实时的调用数据
  - clusterNode: 资源唯一标识的ClusterNode的runtime统计
  - origin: 统计者信息
  - defaultnode: 根据上下文条目名称的资源id的runtime统计
  - 入口的统计
- FlowSlot: 根据流规则进行流量控制。
- AuthoritySlot: 黑白名单
- DegradeSlot: 熔断降级。主要根据资源的平均响应时间以及异常比率，来决定接下来是否熔断降级
- SystemSlot: 控制总的入口流量。根据系统的总体情况对入口资源的调用进行动态调配，让系统预计容量和系统的实际容量达到一个动态平衡

除了上述的核心slot以外，我们也可以自定义slot来实现自己想要的功能

#### 资源与规则

- 资源: 从上面介绍来说，资源是sentinel中基础部分。资源相当于Java中的对象，但是又比对象包含范围大，它可以是一段程序、服务，我们可以定义规则保护它。
- 规则: 资源的保护依赖于规则。
   1. 规则是什么: 规则是对我们定义资源的保护机制，一般我们需要关注资源是否需要保护，具体规则的定制要根据具体内容来选取就可以
   2. 规则有哪些: 
      1. 流量控制规则: FlowRule
      2. 熔断降级规则: DegradeRule
      3. 系统保护规则: SystemRule
      4. 来源访问规则: AuthorityRule
      5. 热点参数规则: ParamFlowRule
   3. 规则的定义: 不同的规则都有对应的规则定义；
   4. 资源绑定:

{% codeblock lang:java %}
// 初始化资源
public static void initRule() {
    FlowRule flowRule = new FlowRule();
    // 资源名
    flowRule.setResource("tmp");
    // 限流阈值
    flowRule.setCount(3);
    // 限流阈值类型
    flowRule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
    // 流控针对的调用来源，default代表不区分
    flowRule.setLimitApp("default");
    List<FlowRule> list = Collections.singletonList(flowRule);
    // 加载规则
    FlowRuleManager.loadRules(list);
}
{% endcodeblock %}

#### 系统自适应保护
原始保护方式: 
- 规则: 通过一个系统的负载来做过载保护，如果高于它则进行限流，低于它进行恢复。
- 弊端:
    - 延迟性: 负载load需要通过一定时间才能反应，不及时
    - 恢复慢: 上下游load不统一情况下，下游高，上游可能还是低，不能根据下游进行及时调整

TCP BBR思想下的保护方式:
- 规则: 根据系统能够处理的请求和允许进来的请求做平衡，最终目的是系统能够在正常允许的基础下，提高吞吐量
- 实现: 通过负载值为启动控制流量的开关，允许通过的流量由请求的平均响应时间和系统处理的请求速率来决定

系统规则:
- 标准: 应用级别的入口流量控制，系统的保护规则是**针对应用整体纬度的，而不是资源纬度**，并且**仅对入口流量生效**。
- 阀值标准: 从单台机器的如下几个方面来考量
  - Load: 系统load超过阀值，并且系统的并发数超过系统容量触发系统保护。系统容量计算: minQps * minRt。参考值: CPU cores * 2.5
  - CPU usage: 系统CPU使用率超过阀值
  - RT: 入口流量的平均rt达到阀值
  - 线程数: 入口流量的并发线程数达到阀值
  - 入口QPS: 入口流量的QPS达到阀值

### Spring Cloud Alibaba Sentinel

1. 引入对应的`starter`
2. 资源注解`@SentinelResource`
- `value`: 资源名
- `blockHandler`: block exception处理类名，访问范围是public，返回类型需要与原方法相同，参数类型也需要与原方法匹配，并最后加一个额外的参数，类型为BlockException。默认需要与原方法在同一个类
- `blockHandlerClass`: block exception处理类。若定义方法不在原方法类，需要设置这个参数，并且设置对应的方法名，范围public，static方法。
- `fallback`: fallback类名，抛出异常时的备选方案
  - 返回值与原函数相同
  - 与原函数在同一个类
  - 方法参数列表需要和原函数一致，或者可以额外多一个 Throwable 类型的参数用于接收对应的异常
- `fallbackClass`: 具体类
- `defaultFallback`: 通用处理类

#### SentinelRestTemplate注解

我们可以通过注解`@SentinelRestTemplate`的源码来看看`fallback`和`blockHandler`的具体参数

{% codeblock lang:java %}
public class SentinelBeanPostProcessor implements MergedBeanDefinitionPostProcessor {

  private void checkSentinelRestTemplate(SentinelRestTemplate sentinelRestTemplate,
    String beanName) {
    // 检查block类型
    checkBlock4RestTemplate(sentinelRestTemplate.blockHandlerClass(), sentinelRestTemplate.blockHandler(), beanName, SentinelConstants.BLOCK_TYPE);
    // 检查fallback类型
    checkBlock4RestTemplate(sentinelRestTemplate.fallbackClass(),sentinelRestTemplate.fallback(), beanName,SentinelConstants.FALLBACK_TYPE);
    // 检查URLCLEANER类型(暂时不关注)
    checkBlock4RestTemplate(sentinelRestTemplate.urlCleanerClass(),sentinelRestTemplate.urlCleaner(), beanName,SentinelConstants.URLCLEANER_TYPE);
  }

  private void checkBlock4RestTemplate(Class<?> blockClass, String blockMethod, String beanName, String type) {
    // 逻辑判断相关类和方法是否存在，不存在则抛出异常()
    ...
    // 核心部分
    // 参数类型
    Class[] args;
    if (type.equals(SentinelConstants.URLCLEANER_TYPE)) {
      args = new Class[] { String.class };
    }
    else {
      // 参数类型组成数组
      args = new Class[] { HttpRequest.class, byte[].class,
      ClientHttpRequestExecution.class, BlockException.class };
    }
    // 通过spring提供的静态方法获取来获取类的public的静态方法
    Method foundMethod = ClassUtils.getStaticMethod(blockClass, blockMethod, args);
    // 后续就是分类型将类及其方法进行注册
  }
}
{% endcodeblock %}

通过上面的源码分析，我们可以知道应该怎么去配置`blockHandler`或者`blockHandlerClass`，如下：

{% codeblock lang:java %}
public class RestTemplateBlock {
    // block
    public static ClientHttpResponse handlerBlock(HttpRequest httpRequest, byte[] body, ClientHttpRequestExecution execution, BlockException blockException) {
        SentinelClientHttpResponse httpResponse = new SentinelClientHttpResponse();
        return httpResponse;
    }
}
{% endcodeblock %}
