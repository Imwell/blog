---
title: Tomcat启动两次问题
date: 2022-04-20 10:50:48
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/tomcat.jpg
tags:
    - Tomcat
categories:    
    - Tomcat
    - Bug
---
## Tomcat启动两次问题

### 起因
多个项目协同，同事在进行项目断点时发现，在进行一个ws链接的时候，竟然会有两次链接请求。

### 分析
由于项目链接启动是通过子线程去启动，所以考虑是否是因为代码问题，导致创建了大于1个线程，才会出现多个创建链接的问题，
经过排差并非这个问题，所以就对Tomcat启动进行检查，发现是tomcat启动两次导致的。

### 原因
由于是Tomcat的原因，自然就是配置出现问题了，检查Tomcat相关配置。当时为了项目访问路径的原因，重新配置相关项目，配置如下:
```xml
<Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">

    <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
           prefix="localhost_access_log" suffix=".txt"
           pattern="%h %l %u %t &quot;%r&quot; %s %b" />


    <!-- 某个项目路径配置,将项目的访问路径重置为path后的地址 -->
    <Context docBase="C:\tools\apache-tomcat-9.0.58\webapps\XXX" path="" reloadable="true"/>
    
</Host>
```
appBase: 项目配置目录地址，可以配置相对路径，也可以配置绝对路径。上面的就是相对路径。
docBase: 项目配置应用目录，配置也和appBase相同
### 解决
上面的配置，为host节点配置了appBase为webapps，所以启动tomcat时，会加载webapps目录下的所有项目，但是在内部也配置了我们的目标项目，导致项目重新启动了一次，
所以就会重新加载，所以为了解决这个问题，我们可以修改配置，如下：
```xml
<Host name="localhost"  appBase="" unpackWARs="true" autoDeploy="true">

    <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
           prefix="localhost_access_log" suffix=".txt"
           pattern="%h %l %u %t &quot;%r&quot; %s %b" />


    <!-- 某个项目路径配置,将项目的访问路径重置为path后的地址 -->
    <Context docBase="C:\tools\apache-tomcat-9.0.58\webapps\XXX" path="" reloadable="true"/>
    
</Host>
```
我们去掉host配置appBase的目录名，让内部项目自己启动，这样项目就不能自动加载，需要我们在内部对其进行一一配置才能访问
