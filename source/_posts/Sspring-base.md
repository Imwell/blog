---
title: Spring基础
date: 2023-02-24 18:51:40
tags:
   - Java
   - Spring
categories:    
   - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
## Spring基础

### Spring中常见的设计模式
1. 单例模式: bean单例
2. 工厂模式: BeanFactory,ApplicationContext
3. 观察者模式: Event事件驱动
4. 适配器模式: advice和aop增强通知
5. 代理模式: aop使用到了动态代理
6. 模版方法模式: jdbcTemplate等模版方法

### SpringMVC的核心组建
1. Dispatcher
2. HandlerMapping
3. handlerAdapter
4. Handler
5. ViewResolver

### SpringIOC && SpringAopP
#### IoC
1. 控制反转: 作为spring最核心的功能，脱离以往对象的相互依赖，相互耦合的情况，现在通过SpringIOC容器创建，管理每个bean，实现了高内聚，低耦合。
2. bean: ioc容器帮我们管理的对象就是bean，它是通过ioc容器创建，其中包含了一些我们普通对象所不具备的功能
3. 注入方式: 
   - `@Autowired`: 类型注入，spring提供的注解。
   - `@Resource`: 名称注入，jdk提供的注解。
   - `@Qualifier`: 注定注入bean的名称，可以和`@Autowired`配合使用
4. bean的作用域
   - singleton: 单例，但不代表就线程安全，具体情况，具体分析
   - prototype: 每次使用重新创建
   - request: 每个http请求都会产生一个新bean，当前请求内有效
   - session: 每个新session都会产生一个新bean，仅在当前session中有效 
   - websocket: 每个websocket中产生一个新bean

#### AoP
1. 面向切面: 能够将那些与业务代码无关的，却为业务逻辑所共同调用的逻辑或责任封装起来，减少代码复杂度，方便结偶
2. Aspect: 与spring的aop相比，性能更好。aop是通过代理方式实现的，aspect是通过字节码实现。

### Spring事务

spring对事务的管理分为两种:
1. 编程式: 通过代码实现对事务的管理
2. 声明式: 通过注解实现对事务的管理，一般使用这种方式

#### 编程式
通过三组接口实现管理:
1. PlatformTransactionManager: 事务管理平台
2. TransactionDefinition: 事务定义信息
3. TransactionStatus: 事务允许状态

#### 声明式
1. 通过注解`@Transactional`来控制当前方法的事务。
2. 作用范围:
   - 方法: 推荐在方法上使用，public上方法使用
   - 类: 所有的public方法都生效
   - 接口: 不推荐
3. 基于aop实现，会根据是否有实现类来确定使用jdk的动态代理还是cglib
4. 使用注意事项:
   - `@Transactional`注解只有作用到 public 方法上事务才生效，不推荐在接口上使用；
   - 避免同一个类中调用`@Transactional`注解的方法，这样会导致事务失效；
   - 正确的设置`@Transactional`的`rollbackFor`和`propagation`属性，否则事务可能会回滚失败;
   - 被`@Transactional`注解的方法所在的类必须被`Spring`管理，否则不生效；
   - 底层使用的数据库必须支持事务机制，否则不生效；

#### 事务属性
1. 事务传播行为: 业务方法之间的调用，事务如何传播？spring提供了多个选择，默认是如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务
2. 事务隔离级别: 不同于数据的隔离级别，spring提供了五个，多了一个默认级别。如下:
   - ISOLATION_DEFAULT: 默认，对应数据库默认隔离级别
   - ISOLATION_READ_UNCOMMITTED: 对应数据库为提交读
   - ISOLATION_READ_COMMITTED: 对应数据库已提交读
   - ISOLATION_REPEATABLE_READ: 对应数据库可重复读
   - ISOLATION_SERIALIZABLE: 对应数据可串行化
3. 事务超时: 设置超时时间，超过则回滚事务

