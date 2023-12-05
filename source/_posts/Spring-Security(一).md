---
title: Spring-Security(一) 基础理论部分
date: 2023-08-16 22:27:58
tags:
    - Java
    - Spring
    - SpringSecurity
categories:    
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# SpringSecurity
SpringSecurity是Spring官方提供的安全访问控制解决方案，功能强大，高度定制，不需要配置特殊的Java认证和授权服务，通常的使用它就可以了。

SpringSecurity是通过一系列Servlet的Filter来完成权限认证的，它可以运行在任何的Servlet的应用中，它可以不是Spring的应用。

## 架构

### FilterChain

客户端发送请求到服务器，mvc应用中，`Servlet`作为`DispatchServlet`的实现，它能够处理一个的`HttpServletRequest`和`HttpServletResponse`，同时，这期间会有若干个`Filters`。它们有如下作用：
- 防止下游过滤器被调用。在这种情况下，过滤器通常会写入HttpServletResponse。
- 修改使用到下游`Filters`和`Servlet`的`HttpServletRequest`和`HttpServletResponse`

过滤器只影响下游的过滤器和servlet，因此调用每个过滤器的顺序非常重要

### DelegatingFilterProxy

它是作为**Spring容器**和**Servlet容器**的**桥梁**。`Servlet`容器可以独立注册`Filter`s，但它并**不是SpringBean**，这个代理就可以将其注册到Spring中。

除此之外，它还可以延后`Filer`Bean的查找加载。容器需要在`Filter`注册完毕之后才启动。Spring通常使用一个`ContextLoaderListener`来加载Spring bean，直到需要注册过滤器实例之后才会完成

![delegatingfilterproxy.png](/img/delegatingfilterproxy.png)

### FilterChainProxy

Spring Security的Servlet支持包含在`FilterChainProxy`中。它是Spring Security提供的一个特殊的`Filter`，`SecurityFilterChain`可以委派很多`Filter`。它也是一个bean，包含在`DelegatingFilterProxy`中。

它为直接的注册到Servlet容器或者`DelegatingFilterProxy`时，提供了有很多好处。如下:
- 它为所有的Spring Security的Servlet支持提供了一个起点，可以为排查Spring Security的故障提供支持，在FilterChainProxy中添加一个debug进行调试
- 作为Spring Security使用的核心，可以执行非可选任务。例如：可以清楚SecurityContext以避免泄漏。
- 在确定何时应调用 SecurityFilterChain 时提供了更大的灵活性。可以利用RequestMatcher接口根据HttpServletRequest中的任何内容确定调用

### SecurityFilterChain

`FilterChainProxy`使用`SecurityFilterChain`来决定在这个请求中哪些SpringSecurity过滤器被使用。`SecurityFilterChain`中的`SecurityFilter`通常是bean，它是在`FilterChainProxy`中注册的。

每个`SecurityFilterChain`可以有单独的配置且每个都是唯一的。如果有多个相同类型的匹配，选择第一个。

![multi-securityfilterchain.png](/img/multi-securityfilterchain.png)

### Security Filters

通过`SecurityFilterChain`API将Security过滤器插入到`FilterChainProxy`中。过滤器顺序很重要。以下是过滤器的一个排序:

|                       过滤器                       | 作用                                                                                    |
|:-----------------------------------------------:|:--------------------------------------------------------------------------------------|
|         ForceEagerSessionCreationFilter         | 如果HttpSession不存在则强制创建                                                                 |
|             ChannelProcessingFilter             | 确保一个web请求通过规定的通道传递                                                                    |
|        WebAsyncManagerIntegrationFilter         | 集成SecurityContext和WebAsyncManager(异步管理)                                               |
|        SecurityContextPersistenceFilter         | 已废弃:  使用SecurityContextHolderFilter                                                   |
|               HeaderWriterFilter                | 给当前请求新增headers                                                                        |
|                   CorsFilter                    | 请求前置处理跨域问题                                                                            |
|                   CsrfFilter                    | csrf保护                                                                                |
|                  LogoutFilter                   | 包含一系列handlers的登出处理，这些handlers必须实现LogoutHandler接口才行。这下接口包含清理上下文，cookie，token等，然后再进行重定向 |
|    OAuth2AuthorizationRequestRedirectFilter     | 需要引入相关jar包                                                                            |
|     Saml2WebSsoAuthenticationRequestFilter      | 需要引入相关jar包                                                                            |
|            X509AuthenticationFilter             | AbstractPreAuthenticatedProcessingFilter的子类，提供两个接口实现                                  |
|  **AbstractPreAuthenticatedProcessingFilter**   | 参考下面特别说明                                                                              |
|             CasAuthenticationFilter             | 需要引入相关jar包                                                                            |
|         OAuth2LoginAuthenticationFilter         | 需要引入相关jar包                                                                            |
|         Saml2WebSsoAuthenticationFilter         | 需要引入相关jar包                                                                            |
|    **UsernamePasswordAuthenticationFilter**     | 继承AbstractAuthenticationProcessingFilter，实现了校验用户接口，router默认匹配/login，用于表单提交的校验         |
|           OpenIDAuthenticationFilter            | 需要引入相关jar包                                                                            |
|        DefaultLoginPageGeneratingFilter         | 默认的登陆，提供内部的登陆错误页面配置，只有重定向到login页面时使用                                                  |
|        DefaultLogoutPageGeneratingFilter        | 默认的登陆出页面，提供登出的页面配置                                                                    |
|             ConcurrentSessionFilter             | 并发session处理包所需要的过滤器。                                                                  |
|         **DigestAuthenticationFilter**          | 一个http请求的digest请求头，并把结果放入SecurityContextHolder                                        |
|       **BearerTokenAuthenticationFilter**       | 需要引入相关jar包                                                                            |
|          **BasicAuthenticationFilter**          | 处理http请求的BASIC授权请求头，并把结果放入SecurityContextHolder。                                      |
|             RequestCacheAwareFilter             | 如果请求匹配到了缓存，那么负责整合包转请求，并使用这个请求继续进行过滤器链，否则使用原请求request                                  |
|     SecurityContextHolderAwareRequestFilter     | 对ServletRequest进行包装，实现Servlet API的一些接口。                                               |
|            JaasApiIntegrationFilter             | 尝试获得一个jaas对象，然后继续进行过滤链。在集成需要填充JAAS主题的代码时，这很有用                                         |
|         RememberMeAuthenticationFilter          | 如果请求拥有记住功能，那么当请求没有用户校验就可以通过这个过滤器将一个校验的token，并写入context。RememberMeServices是核心功能接口      |
|          AnonymousAuthenticationFilter          | 检测在SecurityContextHolder是否有权限，并进行填充                                                   |
|       OAuth2AuthorizationCodeGrantFilter        | 需要引入相关jar包                                                                            |
|             SessionManagementFilter             | 如果用户已经被授权，对session进行管理，比如: 并发                                                         |
|         **ExceptionTranslationFilter**          | 处理从过滤链上抛出的AccessDeniedException和AuthenticationException异常                             |
|          **FilterSecurityInterceptor**          | 通过过滤器实现对HTTP资源进行安全处理                                                                  |
|                SwitchUserFilter                 | 切换用户处理过滤器负责用户上下文切换                                                                    |


### Handling Security Exceptions

`ExceptionTranslationFilter`作为Security过滤器之一插入到`FilterChainProxy`中。

![exceptiontranslationfilter.png](/img/exceptiontranslationfilter.png)

1. 首先，ExceptionTranslationFilter 调用 FilterChain.doFilter(request， response) 来调用应用程序的其余部分。
2. 如果用户没有通过身份验证，或者是 AuthenticationException，那么启动身份验证。
    - SecurityContextHolder 删除
    - HttpServletRequest 保存在 RequestCache 中。当用户成功进行身份验证时，将使用 RequestCache 重放原始请求
    - AuthenticationEntryPoint 用于从客户端请求凭据。例如，它可能会重定向到一个登录页面或发送一个 WWW-Authenticate header
3. 否则，如果是 AccessDeniedException，则 Access Denied。调用 AccessDeniedHandler 来处理拒绝的访问


### 特殊类

![springsecurity.png](/img/springsecurity_struct.png)

#### GenericFilterBean

```java
public abstract class GenericFilterBean implements Filter, BeanNameAware, EnvironmentAware,
		EnvironmentCapable, ServletContextAware, InitializingBean, DisposableBean
{
    ...
}
```
一个filter的基础实现，用来将配置作为bean处理。并把实际的filter方法留给了子类去实现，子类必须实现。

#### OncePerRequestFilter

filter的基类，保证单个请求在servlet的容器中只处理一次。

它的过滤器提供了一个新的属性`already filtered`，用于确认当前是否已经被过滤，如果已经执行过过滤就跳过，没有的话执行的话，它提供了一个`doFilterInternal()`抽象方法，子类通过这个方法继续执行过滤器链

```java
// doFilter方法的参数
protected abstract void doFilterInternal(
        HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
        throws ServletException, IOException;
```

#### AbstractPreAuthenticatedProcessingFilter

前置的验证处理过滤: 
- 这是一个处理前置授权和校验的过滤器基类，它会假设用户已经被用于外部系统所验证。
- 它的目的不仅仅是从用户中提取必须的信息，还包含校验他们。
- 外部的身份验证可能在请求头上提供了相应的信息用于给前置验证过滤器提取
- 通常与PreAuthenticatedAuthenticationProvider配合使用，用于提供用户的额外信息
- 有两个抽象想法，提供不同的前置方法，用于提取用户信息
  - getPreAuthenticatedCredentials(HttpServletRequest request): 提取凭证。不能返回null
  - getPreAuthenticatedPrincipal(HttpServletRequest request): 提取主要信息

#### AbstractAuthenticationProcessingFilter
    授权处理过滤器

基于浏览器和http授权请求的抽象处理，这个过滤器包含一个authenticationManager属性，它是一个接口，处理一个授权请求，需要子类自己实现具体逻辑。
{% codeblock lang:java %}
public interface AuthenticationManager {
    //尝试身份校验，验证通过返回Authentication对象，否则抛出异常
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
{% endcodeblock %}

它还包含一个requiresAuthenticationRequestMatcher属性，确认请求是否匹配，如果匹配就会拦截请求并尝试进行鉴权。具体的方法它提供了一个抽象方法`attemptAuthentication()`，子方法实现执行具体的授权过程。

{% codeblock lang:java %}
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
    implements ApplicationEventPublisherAware, MessageSourceAware {
    // 执行实际的身份校验。会返回以下其中一个结果:
    // 1. 返回一个身份校验成功的token
    // 2. 返回一个null。流程还在执行，需要子类去执行额外的流程
    // 3. 抛出异常，校验失败的话
    public abstract Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException, IOException, ServletException;
}
{% endcodeblock %}

![abstractauthenticationprocessingfilter](/img/abstractauthenticationprocessingfilter.png)

1. 用户提交凭证，`AbstractAuthenticationProcessingFilter`就会创建一个已经被校验过的`Authentication``，它是依赖`AbstractAuthenticationProcessingFilter`的子类创建的。比如: ` UsernamePasswordAuthenticationFilter`会根据提交的用户名/密码创建一个token
2. `Authentication`被`AuthenticationManager`校验
3. 失败执行
4. 成功执行
   1. session记录
   2. `SecurityContextHolder`中设置`Authentication`，`SecurityContextPersistenceFilter`将`SecurityContext`设置到`HttpSession`
   3. `loginSuccess`执行。如果没配置remember me，则忽略
   4. 发布事件`InteractiveAuthenticationSuccessEvent`
   5. `AuthenticationSuccessHandler`执行

## Servlet身份验证架构

以下部分就是Servlet身份校验的主要组件

- SecurityContextHolder: SpringSecurity存储认证用户信息详情的地方。
- SecurityContext: 从SecurityContextHolder中获得，包含当前已验证用户信息
- Authentication: 可以是AuthenticationManager的入参，用来给用户提供凭证，也可以是来自SecurityContext的当前用户
- GrantedAuthority: 在身份验证时授予主体的权限
- AuthenticationManager: SpringSecurity的过滤器执行授权的api定义
- ProviderManager: AuthenticationManager的大部分的公共实现
- AuthenticationProvider: 被用于ProviderManager中执行特定的授权类型
- Request Credentials with AuthenticationEntryPoint: 客户端的请求凭证
- AbstractAuthenticationProcessingFilter: 用于授权的基础过滤器

### SecurityContextHolder

![securitycontextholder](/img/securitycontextholder.png)

它是SpringSecurity的存储空间，专门存储被校验过的用户。它不在乎写入什么，只决定写入的值被用与当前被校验的用户。
{% codeblock lang:java %}
// 创建一个空的SecurityContext
SecurityContext context = SecurityContextHolder.createEmptyContext();
// 创建一个校验用户
Authentication authentication = new TestingAuthenticationToken("username", "password", "ROLE_USER");
// 然后在上下午中存入被校验用户
context.setAuthentication(authentication);
// 将上下文存入SecurityContextHolder。SpringSecurity将使用这个信息去授权
SecurityContextHolder.setContext(context);
{% endcodeblock %}

默认情况下，SecurityContextHolder是使用一个`ThreadLocal`存储这些信息，这就意味着SecurityContext在相同线程中的方法总是可用的，甚至SecurityContext不需要作为一个显示的参数在方法间进行传递

{% codeblock lang:java %}
// 当前线程中获取上下文
SecurityContext context = SecurityContextHolder.getContext();
// 获取授权信息
Authentication authentication = context.getAuthentication();
String username = authentication.getName();
Object principal = authentication.getPrincipal();
// 权限列表
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
{% endcodeblock %}

### Authentication

SpringSecurity在身份校验主要有两个目的:
- 作为AuthenticationManager的参数，给一个用户提供身份校验的凭证。
- 代表当前已经校验过的用户

它包含三个内容:
- principal: 主体。用户的主要信息，如果是通过用户名与秘密的验证方式，那么返回的会是一个UserDetails的实力
- credentials: 凭证。通常是一个密码，在验证之后会将它清楚，为了避免泄漏
- authorities: 权限。用户被授予的高级别的权限。

### GrantedAuthority

GrantedAuthority是赋予用户的高级别的权限。比如: 角色或者作用域。

### ProviderManager

>ProviderManager通常是AuthenticationManager的实现。ProviderManager也委托了一系列的AuthenticationManager。
每个AuthenticationManager都有一个机会表明验证是成功还是失败，或者表明它需要后续流程的AuthenticationProvider去决定。
如果没有相对应的AuthenticationProvider配置能够校验，就会抛出一个异常，表明没有配置支持的校验类型

事实上，多数的`ProviderManager`它们拥有相同的父类`AuthenticationManager`，在多数的`SecurityFilterChain`实例中就有相同的验证方法，但同时也会有不同的`ProviderManager`实例实现的验证

通常，默认的`ProviderManager`会在成功的校验请求中的尝试清空凭证的敏感信息。

### AuthenticationProvider

大多数`AuthenticationProvider`能注入`ProviderManager`，每个`AuthenticationProvider`都会执行一个特别的校验。

### AuthenticationEntryPoint

`AuthenticationEntryPoint`被用于发送一个http的响应，从客户端请求凭证。通常，客户端都会主动包含相关的凭证，如果客户端不存在请求的权限，一个`AuthenticationEntryPoint`的实现就会去请求凭证。这个实现可能会执行一个重定向，并回应一个带有www-authenticate的响应

