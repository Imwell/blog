---
title: SpringBoot实现接口加密和解密
date: 2023-07-28 10:31:57
tags:
  - Java
  - Spring
categories:
  - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# SpringBoot实现接口加密和解密

日常使用不可避免一些特别的业务场景，需要我们对数据进行保密控制，一般使用aop对接口进行加密&解密比较耦合，不利用后续服用，我们可以创建一个starter项目，提供加密解密功能

## 核心点
- 自定义starter功能
- 拦截请求进行解密
- 拦截响应进行加密
- 请求数据流多次读取

## 项目创建

![项目预览](/img/springboot-encrypt.png)

- advice目录: 拦截响应或者请求并更具注解进行加密&解密
- config目录：自动配置及其额外的自定义配置项
- entity目录: 定义两个实体类，在starter引入的项目中的，对加密和解密进行一些特殊字段的定义

### request请求流处理

- **问题**: 在接口调用链路中，request的请求流只能调用一次，处理之后，如果后续还想用到请求流，就会发现数据为空。
- **为什么会这样**: 传统的http请求流中，请求流是一个单向流动的过程，一旦读取之后，就不能再次重复读取了。这是由于http协议的设计和工作原理所决定的
- **解决方法**: 既然http只能读取一次，我们可以将它缓存到另一个可以重复读取的流中，让我们任意时刻进行读取
- **具体解决方法如下**：
  - 继承`HttpServletRequestWrapper`类，重写`getInputStream()`方法和`getReader()`方法。
  - 在其中实现copy请求流的逻辑，那么以后调用这两个方法就都是从copy的流中拿取流数据。
  - 配合filter过滤器，在一开始就进行替换request中的流

{% codeblock lang:java %}
// 继承
public class InputStreamHttpServletRequestWrapper extends HttpServletRequestWrapper {
    // 缓存流
    private ByteArrayOutputStream cacheBytes;

    // 重写获取输入流
    @Override
    public ServletInputStream getInputStream() throws IOException {
        // 判断缓存流
        if (cacheBytes == null) {
            cacheInputStream();
        }
        return new CacheServletInputStream(cacheBytes.toByteArray());
    }

    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(getInputStream()));
    }

    private void cacheInputStream() throws IOException {
        cacheBytes = new ByteArrayOutputStream();
        // 拷贝原request流到自定义的缓存流
        IOUtils.copy(super.getInputStream(), cacheBytes);
    }
    
    // 读取缓存的请求正文的输入流
    static class CacheServletInputStream extends ServletInputStream {
        // 存储流
        private final ByteArrayInputStream arrayInputStream;
        // 构造器赋值
        public CacheServletInputStream(byte[] bytes) {
            this.arrayInputStream = new ByteArrayInputStream(bytes);
        }

        @Override
        public int read() throws IOException {
            return arrayInputStream.read();
        }
        // 剩下的根据实际情况实现
    }
}
{% endcodeblock %}

### 加密 & 解密

- 加解密的方法
  - 使用**AES对称加密**，这种加密相对应非对称加密更方便快捷，为了避免被破解，我们可以通过接口获取动态密钥的方式，然后进行前端加密后端解密校验。
  - 这里我们使用了hutool工具的加密包进行加密，通过方便起见，我们直接通过配置文件来配置密钥和盐值。

- 请求数据中加解密
  - 注解`@ControllerAdvice`: 定义该类为SpringBean，可以自动扫描类路径
  - 请求解密: 继承`RequestBodyAdvice`类，在读取请求体并将其转换为对象前，对其进行处理
    - supports(): 确定哪些方法需要解密
    - afterBodyRead(): 在body进行转换转换处理之后再进行处理
  - 请求加密: 继承`ResponseBodyAdvice`类，在标注了注解`@ResponseBody`的方法控制器执行
    - supports(): 确定哪些方法需要解密
    - beforeBodyWrite(): HttpMessageConverter方法的writer方法调用之前执行

### entity实体类

我们定义了两个实体类，`RequestBase`和`RequestData`
- RequestBase: 用来给需要加密的实体类是否需要增加额外的一些字段，用来进行判断具体的业务情况，比如时间戳，用来判断加解密的时间范围
- RequestData: 定义了前端加密之后提交的post数据结构，方便我们对其中数据进行解密

### 定义自动配置

1. 在自动配置类上加如下注解，
2. 在`resources`的目录`MATE-INF`下加`spring.factories`文件，内容为:`org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.star.crypto.config.CryptAutoConfiguration`
3. 打包发布进行测试

{% codeblock lang:java %}
@Configuration
@EnableConfigurationProperties(CryptProperties.class)
// 一定要加行扫描注解，选定功能范围
@ComponentScan("org.star.crypto")
public class CryptAutoConfiguration {
    @Resource
    private CryptProperties cryptConfig;

    @Bean
    public AES aes() {
        return new AES(cryptConfig.getMode(), cryptConfig.getPadding(), cryptConfig.getKey().getBytes(StandardCharsets.UTF_8), cryptConfig.getIv().getBytes(StandardCharsets.UTF_8));
    }

    /**
     * 将filter注入
     * @return
     */
    @Bean
    public HttpServletRequestInputStreamFilter httpServletRequestInputStreamFilter() {
        return new HttpServletRequestInputStreamFilter();
    }
}

{% endcodeblock %}