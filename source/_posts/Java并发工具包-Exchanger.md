---
title: Java并发工具包-Exchanger
date: 2021-03-08 09:51:56
tags:
    - Java
    - 并发
categories:
    - 并发
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
# Exchanger
交换器，从字面意思上理解，它就是对两个线程的数据进行交换的，通过这个工具类，简化了线程之间数据交换的成本，并且提供了数据交换点。

{% codeblock lang:java %}
    
    // 创建字符串类型的数据交换器 
    final Exchanger<String> stringExchanger = new Exchanger<>();
    // 创建四个线程
    for (int i = 0; i < 4; i++) {
        new Thread(() -> {
    
            System.out.println(Thread.currentThread().getName() + " start");
    
            try {
                TimeUnit.MILLISECONDS.sleep(20);
                String exchange = stringExchanger.exchange(Thread.currentThread().getName() + "发送的");
                System.out.println(Thread.currentThread().getName() +"接收到：" + exchange);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "X" + i).start();
    }
{% endcodeblock %}

`exhange()`方法是一个阻塞方法，一个线程调用了之后就会进入阻塞，必须在另一个线程再次调用了才会推出阻塞。
**它用于数据交换的前提是，exchanger方法被成对调用**

# Exchange方法
`public V exchanger(V x) throw InterruptedException`：数据交换方法，将数据交换至搭档线程

`public V exchanger(V x, long timeout, TimeUnit unit) throw InterruptedException`：用法如上，增加了超时功能

