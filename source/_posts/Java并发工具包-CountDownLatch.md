---
title: Java并发工具包 CountDownLatch
date: 2021-02-08 17:41:49
tags:
    - Java
    - 并发
categories:
    - 并发
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
# CountDownLatch
倒计数门阀。一般用在多任务的情景中。一个主任务，多个副任务，而且需要得到所有副任务的结果，进而才能继续进行主任务。CountDownLatch是一个同步助手，允许一个或多个线程等待一系类的其他线程执行结束。
通过观察源码，使用底层的是抽象的队列式同步器（**AQS**）。
# Latch(门阀)设计模式
该模式指定了一个屏障，只有所有的条件都得到满足，屏障才能取消，门阀才能打开。
# 常用方法
- CountDownLatch(int count)：构造方法，设置count数量，不能小于0
- countDown()：构造函数指定的count减1，如果此时为0，那么下次调用就会被忽略，最小只能为0
- await()：使得当前调用进入阻塞状态，直到count为0。当count为0时调用，会直接返回，当前线程不再阻塞
- await(long timeout, TimeUnit unit)：为阻塞设置超时，防止一直阻塞
- getCount()：获取当前count计数的值

# 常见场景
对某些商品进行计算最终价格，由于商品涉及的模块可能不只有一个，可以采取两种方案：
- 串行执行，对每个商品进行一步步的任务执行，最终得到答案
- 并行执行，如果商品数量有限，进行多线程任务处理，每个商品作为一个线程，最终得到答案

以上的两种方案，明显第二种的效率高，那么，我们就可以通过使用**CountDownLatch**工具，来让任务并行执行，如下所示：
{% codeblock lang:java %}
public static void main(String[] args) throws InterruptedException {

    // 获取list集合
    List<Goods> list = Goods.getGoodsList();
    // 设置门阀初始值，不能小于0
    final CountDownLatch downLatch = new CountDownLatch(list.size());
    // 循环创建线程
    list.forEach(g -> {
        new Thread(() -> {
            try {
                // 模拟业务逻辑
                TimeUnit.MILLISECONDS.sleep(20);
                g.setThreadContent(Thread.currentThread().getName());
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                downLatch.countDown(); // 正常执行与否，都需要调用，使count数量-1
            }

        }).start();
    });
    // 主线程阻塞，等待子线程执行完所有任务
    downLatch.await();
    System.out.println("all of goods has finished");
    list.forEach(System.out::println);
}

{% endcodeblock %}
