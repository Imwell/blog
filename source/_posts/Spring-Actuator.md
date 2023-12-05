---
title: Spring Actuator
date: 2021-01-29 20:49:30
tags:
    - Java
    - Spring
categories:
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
# Actuator
在机器领域中，执行机构(Actuator)指的是负责控制流和移动装置的组件。在SpringBoot中，角色性质也是一致的，它主要负责组件的内部运行状态，能够在一些程度上控制应用行为。
通过Actuator，我们可以获得的应用内部信息如下：
- 应用环境中的配置信息
- 各个源码包的日志级别
- HTTP访问请求次数
- 请求的外部服务健康状况
- 应用消耗的内存

## 配置启用
我们可以直接在POM文件中写入依赖
```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```
默认情况下，我们启动应用，通过url去访问，只能够访问到health，而且也是没有任何信息的，默认的监控配置如下：

| ID | JMX | Web |
|:---:|:---:|:---:|
|auditevents|Yes|No|
|beans|Yes|No|
|caches|Yes|No|
|conditions|Yes|No|
|configprops|Yes|No|
|env|Yes|No|
|flyway|Yes|No|
|health|Yes|No|
|heapdump|N/A|Yes|
|httptrace|Yes|No|
|info|Yes|No|
|integrationgraph|Yes|No|
|jolokia|N/A|No|
|logfile|N/A|No|
|loggers|Yes|No|
|liquibase|Yes|No|
|metrics|Yes|No|
|mappings|Yes|No|
|prometheus|N/A|No|
|quartz|Yes|No|
|scheduledtasks|Yes|No|
|sessions|Yes|No|
|shutdown|Yes|No|
|startup|Yes|No|
|threaddump|Yes|No|
