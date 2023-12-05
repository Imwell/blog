---
title: Nginx301重定向Bug
date: 2022-06-24 18:00:13
tags:
    - NGINX
    - BUG
categories:
    - NGINX
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/nginx.jpg
---
# Nginx 301跳转问题

## 起因
公司因为成本问题缩减服务器数量，个别服务因为使用频率比较低，而且权重也不高，所有决定将其缩减到一个服务器当中。
但是，在一个服务器上就会有个问题，443和80端口只有一组，我们要兼容多个服务的域名，程序内部当然做不到。
但是可以采取不同端口来实现，再让运维把域名的443端口转发到机器对应服务的端口上，这个问题就解决了。
不过后来又出现了新的问题，通过https域名访问前端项目，首页是正常的，切换路由就出现301问题，观察network，
可以看到在response中又location重定向，而且重定向指向到`http+端口号+路由`的地址，域名并没有开通对应端口号的权限，最后就出现了访问失败的情况。

## 分析
1. 3xx状态码：http协议中3xx都表示重定向的响应。301是代表永久重定向。
2. 触发条件：用户访问一个路由，并不以`/`结尾，例如：https://xxx.xxxx.com/backstage, backstage这个文件没有找到，但是发现这是个目录，于是这次访问就会被设置为301进行跳转。
3. **Nginx负责设置301 Moved Permanently状态码。但nginx.conf控制Nginx如何处理301 Moved Permanently状态码！ 换句话说，要不要进行页面重定向，和怎么重定向，完全是用户配置的结果！**

## 解决
既然知道了这是由用户配置的，那我们就可以先查找其配置项，了解到跳转的逻辑。
相关配置：absolute_redirect ，server_name_in_redirect和port_in_redirect。

- absolute_redirect设置成on，则生成绝对路径作为Location url。
- absolute_redirect设置成off，则生成相对路径作为Location url。

只有absolute_redirect设置为on时，另外两个配置才会生效。

- server_name_in_redirect为on时，使用Nginx配置文件中的server_name作为Location url中的host，否则使用用户请求url中的主机名作为host；
- port_in_redirect设置为on时，使用nginx监听的端口来构造Location url，否则不设置port。

上述三个配置共同控制了Location url的结果。例如：Location: http://xxx.xxxx.com:port/backstage/
三项的默认值为absolute_redirect=on,server_name_in_redirect=off,port_in_redirect=on

那么，梳理下原因，nginx设置了301状态码，并设置http协议的Location，url为绝对路径+端口号+路由，浏览器接收到响应并进行了解析，出现了问题。
最简单的解决方法就是把重定向的绝对路径改为相对路径，构造相对路径的url，Location的值就只有路由，跳转就正确了。

![img.png](/img/nginx-bug-301.png)

参考链接： https://juejin.cn/post/7021818339651485726
