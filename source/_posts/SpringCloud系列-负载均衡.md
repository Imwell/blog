---
title: SpringCloud系列-负载均衡
date: 2021-02-09 19:26:14
tags:
    - Java
    - SpringCloud
categories:
    - SpringCloud
    - 负载均衡
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# 负载均衡
负载均衡，是一种计算机的技术，用来在多个计算机，网络连接，CPU，磁盘驱动器或其他资源中分配负载，已达到资源利用率最高，吞吐率最大换，最小化响应时间，同时避免过载的问题。（wiki）

在我们常见的微服务的架构当中，注册中心负责服务的注册和发现，但是，它也只能作为服务的提供者，如果我们需要获取某个服务，可以通过注册中心提供的Api获取对应的服务，但是，一般都会返回一个服务列表，所以，我们需要从中选取其中一个，作为我们需要的实例返回。这就需要一个新的算法来解决选取这一过程--负载均衡。

常见的负载均衡算法：
- 随机算法：随机选取
- 轮训算法：一个一个轮询选取
- 最少连接数算法：选取连接数最小的是咧
- 一致性哈希算法：
- 权重随机算法：不同实例权重不同，看随机数落在那个权重范围内，则选取哪个

负载均衡是一种通用的特性，所有的RPC框架都会有这个概念的实现。Apache Dubbo使用的是Load Balancer组件来实现的，Spring Cloud使用的是Nexflix Ribbon，并在将来用SpringCloud LoadBalancer（SCL）来作为替代品。SCL虽然已经问世，但是目前功能不齐全。
{% codeblock lang:java %}
// 获取服务的所有实例
List<ServiceInstance> instances = client.getInstances(serviceName);
// 随机选择一个
ServiceInstance serviceInstance = instances.stream().findAny().orElseThrow(() -> new IllegalStateException("no " + serviceName + " instance available"));
{% endcodeblock %}

# SpringCloud LoadBalancer
相关依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```
核心接口和类：
- ServiceInstanceChooser：服务实例选择器，根据服务名选择一个服务实例，接口
- LoadBalancerClient：客户端负载均衡器，继承了ServiceInstanceChooser，会根据选取的实例和Request请求执行最终的结果
- BlockingLoadBalancerClient：基于SpringCloud LoadBalancer的默认实现
- RibbonLoadBalancerClient：基于SpringCloud Netflix的默认实现

## RestTemplate/WebClient
RestTemplate/WebClient是一种简单、易用的Java客户端，我们可以基于它进行服务间的调用。具体方法可以参考文档，不再赘述。

## 基于实例的ip和端口调用
由于ServiceInstanceChooser是接口，我们就可以自己去实现这个接口，自定义获取服务实例的随机算法。
{% codeblock lang:java %}

@Service
public class RandomLoadBalancerService implements ServiceInstanceChooser { // 实现ServiceInstanceChooser接口，自定义choose算法

    private final DiscoveryClient discoveryClient;

    private final Random random;

    public RandomLoadBalancerService(DiscoveryClient discoveryClient) {
        this.discoveryClient = discoveryClient;
        this.random = new Random();
    }

    @Override
    public ServiceInstance choose(String serviceId) {
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
        return instances.get(random.nextInt(instances.size())); // 使用随机获取其中一个实例
    }
}
{% endcodeblock %}

## 基于服务名调用
通过服务名调用，我们只需要知道对应的服务名就可以了，不需要再去获取对应服务的实例信息，但是，由此又会有新的问题出现，怎么才能实现负载均衡算法呢？
其实SpringCloud自己就包含了负载均衡的操作，我们可以加入注解`@LoadBalanced`在RestTemplate的Bean对象上，这样，我们就能通过服务名调用，而且不用再去处理负载均衡的问题
{% codeblock lang:java %}

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
{% endcodeblock %}

## @LoadBalanced
注解`@LoadBalanced`到底做了什么事，我们可以通过观察源码来了解下。如下是LoadBalancer的自动配置文件。

{% codeblock lang:java %}

/**
 * 自动注入配置
 * Auto-configuration for Ribbon (client-side load balancing).
 */
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class) // 只有存在RestTemplate类的时候才能被加载
@ConditionalOnBean(LoadBalancerClient.class) // LoadBalancerClient的Bean存在时才能被加载
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
   

    /*
     * 把所有使用了@LoadBalanced注解的Bean加载进集合中
     */
    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();
    
    @Autowired(required = false)
    private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();
    
    /*
     * SmartInitializingSingleton接口：Spring容器把所有的单例Bean都初始化完成之后，会调用这个接口。
     * ObjectProvider接口：ObjectFactory的扩展接口，专门为注入点设计，可以让注入变得更加宽松和更具有可选性
     *
     * 这里会对所有的RestTemplate做一个定制处理
     */
    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
            final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) { // 注入List<RestTemplateCustomizer>
        return () -> restTemplateCustomizers.ifAvailable(customizers -> { // 判断存在，如果存在，就执行lamda
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) { // 循环模板进行定制处理
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
    }

    /*
     * 注入LoadBalancerRequestFactory，通过一个负载均衡客户端和集合，主要用来构造LoadBalancerRequest
     */
    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
    }


    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    static class LoadBalancerInterceptorConfig {
    
        /*
         * 注入LoadBalancerInterceptor拦截器
         */
        @Bean
        public LoadBalancerInterceptor ribbonInterceptor(
                LoadBalancerClient loadBalancerClient,
                LoadBalancerRequestFactory requestFactory) { // 上面注入LoadBalancerRequestFactory
            return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
        }
    
        /*
         * 注入定制的Bean，上面循环的定制操作就是在这里
         * 注：如果自由有实现RestTemplateCustomizer的Bean，那么这里的就不会生效
         */
        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(
                final LoadBalancerInterceptor loadBalancerInterceptor) { // 上面注入LoadBalancerInterceptor拦截器
            return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor); // 在集合中添加拦截器，并放入RestTemplate中，定制结束，RestTemplate就具有了负载均衡的功能
                restTemplate.setInterceptors(list);
            };
        }
    }
}

{% endcodeblock %}

**总结**：让RestTemplate获得负载均衡功能的主要是因为一个拦截器
1. LoadBalancerAutoConfiguration配置的有条件的自动注入
2. @Autowired将所有的标注了@LoadBalanced的Bean放入集合，为后续定制
3. 注入LoadBalancerRequestFactory，LoadBalancerInterceptor的Bean的Bean
4. 获取所有集合，并对RestTemplate进行循环和定制
5. 将LoadBalancerInterceptor的Bean拦截器放入RestTemplate中，作为定制内容
6. RestTemplate获得负载均衡功能


上面展示了LoadBalancer的自动配置文件，具体说明了RestTemplate的定制过程，就是这个过程让其拥有了负载均衡的功能。那么，接下来我们来看看负载均衡的流程实在哪里实现的。
其实，最主要的就是一个拦截器类**LoadBalancerInterceptor**

{% codeblock lang:java %}

public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

    private LoadBalancerClient loadBalancer;
    
    private LoadBalancerRequestFactory requestFactory;
    
    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer,
        LoadBalancerRequestFactory requestFactory) {
        this.loadBalancer = loadBalancer;
        this.requestFactory = requestFactory;
    }
    // ... 省略几行代码
    
    /*
     * 重写父类方法intercept()，对每个请求做拦截处理，并返回一个Response。
     * 看到这，我们也就知道了，负载均衡就是在这处理的。
     */
    @Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        Assert.state(serviceName != null,
                "Request URI does not contain a valid hostname: " + originalUri);

        // 使用loadBalancerClient的execute方法，真正做请求的地方，使用了服务名和LoadBalancerRequest
        return this.loadBalancer.execute(serviceName,
                // 构建LoadBalancerRequest，返回一个request的对象，并用这个request来做真正的请求
                this.requestFactory.createRequest(request, body, execution));
    }

}

public class LoadBalancerRequestFactory {

    // ... 省略其他代码    
    
    /*
     * 构造LoadBalancerRequest。
     * LoadBalancerRequest是一个接口，这里使用了匿名内部类返回了它的实现对象，从中我们可以看到，
     * 在①处，通过request请求，外部传入的服务实例instance和LoadBalancerClient，
        构造出了HttpRequest的实现类ServiceRequestWrapper，其中基本信息都已经填充完毕
     * 在②处，根据LoadBalancerRequestTransformer对请求在做二次加工
     * 在③处，返回requst实例，并交给LoadBalancerClient的excute()方法，做请求
     */
    public LoadBalancerRequest<ClientHttpResponse> createRequest(
            final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) {
        return instance -> {
            HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, // ①
                    this.loadBalancer);
            if (this.transformers != null) { // ②
                for (LoadBalancerRequestTransformer transformer : this.transformers) {
                    serviceRequest = transformer.transformRequest(serviceRequest,
                            instance);
                }
            }
            return execution.execute(serviceRequest, body); // ③
        };
    }
}
{% endcodeblock %}

{% codeblock lang:java %}

public interface LoadBalancerClient extends ServiceInstanceChooser {

    // 执行请求，request就是通过LoadBalancerRequestFactory构造的
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
    <T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException;
    // 重新构造url：把url中原来写的服务名 换掉 换成实际的
    URI reconstructURI(ServiceInstance instance, URI original);
}
{% endcodeblock %}

**总结**：
这部分主要拦截器中的实现，从代码可以看到，请求操作是在拦截器方法里面使用LoadBalancerClient执行的。LoadBalancerRequestFactory中构造的request才是LoadBalancerClient执行请求的request。

## BlockingLoadBalancerClient
LoadBalancerClient的默认的实现类是基于SCL的BlockingLoadBalancerClient。其中实现了具体请求的逻辑。

{% codeblock lang:java %}

public class BlockingLoadBalancerClient implements LoadBalancerClient {
    private final LoadBalancerClientFactory loadBalancerClientFactory;

    public BlockingLoadBalancerClient(LoadBalancerClientFactory loadBalancerClientFactory) {
        this.loadBalancerClientFactory = loadBalancerClientFactory;
    }
    
    /*
     * 没有传入服务的实例，首先会去获取serviceInstance，然后调用重载方法，去执行请求。实例为空则抛出异常
     */
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
        ServiceInstance serviceInstance = this.choose(serviceId);
        if (serviceInstance == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        } else {
            return this.execute(serviceId, serviceInstance, request);
        }
    }

    /*
     * 调用传入的LoadBalancerRequest的方法，从之前的代码得出，这个调用就是请求那个匿名内部类，传入实例作为参数，来执行负载均衡的Http请求
     */
    public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
        try {
            return request.apply(serviceInstance);
        } catch (IOException var5) {
            throw var5;
        } catch (Exception var6) {
            ReflectionUtils.rethrowRuntimeException(var6);
            return null;
        }
    }

    public URI reconstructURI(ServiceInstance serviceInstance, URI original) {
        return LoadBalancerUriTools.reconstructURI(serviceInstance, original);
    }

    public ServiceInstance choose(String serviceId) {
        ReactiveLoadBalancer<ServiceInstance> loadBalancer = this.loadBalancerClientFactory.getInstance(serviceId);
        if (loadBalancer == null) {
            return null;
        } else {
            Response<ServiceInstance> loadBalancerResponse = (Response)Mono.from(loadBalancer.choose()).block();
            return loadBalancerResponse == null ? null : (ServiceInstance)loadBalancerResponse.getServer();
        }
    }
}

{% endcodeblock %}

## @Qualifier的特殊用法
在LoadBalancerAutoConfiguration的代码中，有如下一段代码，它能把容器内所有RestTemplate类型并且标注有@LoadBalanced注解的Bean全注入进来。
从注解`@LoadBalanced`的代码中发现，这个注解类上有一个`@Qualifier`注解，这个注解用于"精确匹配"Bean，一般用于同一类型的Bean有多个不同实例的case下，可通过此注解来做鉴别和匹配。
该文章中有很详细的解答，我之后再研究下。https://fangshixiang.blog.csdn.net/article/details/100890879

{% codeblock lang:java %}

    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

{% endcodeblock %}

{% codeblock lang:java %}
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {

}
{% endcodeblock %}
