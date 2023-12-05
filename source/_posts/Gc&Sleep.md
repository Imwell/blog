---
title: Sleep和GC的关系
date: 2022-09-22 10:09:06
tags:
- Java
categories:    
- Java
description: 初始化垃圾清理
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
# Sleep和GC的关系
## 起因
一切都是从一段代码和注释开始的
{% codeblock lang:java %}
for (int i = 0, j = 0; i < 10000; i++, j++) {
    // 业务逻辑
    // ...
    // prevent gc
    if (j % 1000 == 0) {
        System.out.println("Test");
        try {
            Thread.sleep(0);
        } catch (InterruptedException exception) {
            System.out.println(exception.getMessage());
        }
    }
}
{% endcodeblock %}
1. 从代码看，这是个大循环，而且里面会有一个每循环1000就去执行的逻辑
2. if整体代码不难，但是if里面的代码确有点让人费解，sleep(0)的真正意图是什么
3. 从注释来看是为了防止GC，这段代码出处是RocketMQ，如下图:
![img.png](/img/gc_sleep.png)
> org.apache.rocketmq.store.logfile.DefaultMappedFile#warmMappedFile

## 探究
起因已经了解了，那么这段代码的真实含义和注释是否相同呢？那么可以从issue种寻求答案，具体看看是什么情况下才会写sleep(0)。
从这个[答案](https://stackoverflow.com/questions/53284031/why-thread-sleep0-can-prevent-gc-in-rocketmq)可以初步了解，是可以让JVM可以更快进行GC,防止长时间之后再进行GC，也就是说，并不是防止GC的，反而是要触发GC。
**那么，触发GC有什么条件呢？** 安全点

### 安全点
有了安全点的设定，也就决定了用户程序执行时并非在代码指令流的任意位置都能够停顿下来开始垃圾收集，而是强制要求必须执行到达安全点后才能够暂停。
> 是 HotSpot 虚拟机为了避免安全点过多带来过重的负担，对循环还有一项优化措施，认为循环次数较少的话，执行时间应该也不会太长，所以使用 int 类型或范围更小的数据类型作为索引值的循环默认是不会被放置安全点的。这种循环被称为可数循环（Counted Loop）
> 相对应地，使用 long 或者范围更大的数据类型作为索引值的循环就被称为不可数循环（Uncounted Loop），将会被放置安全点。

也就是说，可数循环会在循环结束之后才进入安全点，不可数循环可以在循环时候进入安全点。

从上面RocketMQ代码看，为了进入安全点进行GC，**那么sleep(0)是如何完成进入安全点？**

从safepoint的cpp[源码](https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/tip/src/share/vm/runtime/safepoint.cpp)注释看，这句话就解释了原因。
![img.png](/img/safepoint.png)

正好，`Thread.sleep`源码就是native修饰的,那么，调用sleep就会进行检测安全点，那么，进入安全点，从而进行GC。
