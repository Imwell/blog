---
title: ScheduledExecutorService
date: 2022-10-08 20:27:03
tags:
 - Java
categories:
 - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
# Java自带的定时任务调度 - ScheduledThreadPoolExecutor

目前如果使用简单的定时任务调度，我们会采用注解`@Scheduled`的方式，这个方式简单易懂，而且不需要太多的操作。
但是，也因为如此，在一些场景下就不符合需求了，那么如果采用定时任务框`quartz`，那么简单操作就又复杂化了，这时候，使用`ScheduledThreadPoolExecutor`，就成了我们的折中方案。

## ScheduledThreadPoolExecutor
    
设计出的目的就是作为延时或者定期执行，而且可以看到，我们可以把任务分发到子线程处理，完美利用到了CPU，如果不是需要定制特殊的定时任务，那么这个就能满足绝大多是任务场景了。

它主要包含有四个方法：
{% codeblock lang:java %}
// 带有延迟的执行一次任务，并通过Future.get()阻塞任务直到执行结束
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
// 带有延迟的执行一次任务，并通过Future.get()阻塞任务直到执行结束，并获取任务结果
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);
// 带延迟时间的调度，循环执行，固定频率
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);
// 带延迟时间的调度，循环执行，固定延迟
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);

{% endcodeblock %}

## 基本使用

### schedule Runnable
{% codeblock lang:java %}
// begin: 2022-10-10 02:50:54
// 2022-10-10 02:50:57 schedule Runnable
// 从上面时间点来看，是3s延迟之后执行
void runnable() {
    ScheduledFuture<?> schedule = executorService.schedule(() -> System.out.println("schedule Runnable"),
        3, TimeUnit.SECONDS);
    try {
        System.out.println(getDate() + " runnable result: " + schedule.get()); // get()会返回null
    } catch (InterruptedException | ExecutionException e) {
        throw new RuntimeException(e);
    }
}

{% endcodeblock %}

### schedule Callable
// begin: 2022-10-10 02:50:57
// 2022-10-10 02:51:03 callable result: callable
// 从上面时间点来看，是1s延迟之后再执行callable，业务里面延迟5s之后再通过get()方法拿到结果，get()再没有拿到结果都是阻塞的
{% codeblock lang:java %}
void callable() {
    ScheduledFuture<String> schedule = executorService.schedule(() -> {
        Thread.sleep(5000);
        return "callable";
    }, 1, TimeUnit.SECONDS);
    String s = null;
    try {
        s = schedule.get();
    } catch (InterruptedException | ExecutionException e) {
        throw new RuntimeException(e);
    }
    System.out.println(getDate() + " callable result: " + s);
}

{% endcodeblock %}
