---
title: Java并发工具包 CyclicBarrier
date: 2021-02-08 17:43:28
tags:
    - Java
    - 并发
categories:
    - 并发
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
# CyclicBarrier
循环屏障，也是一个同步助手工具，它允许多个线程在执行完某些操作之后，到达某个障点（barrier point）进行阻塞，当分片数为0时，再继续进行之后的操作。和CountDownLatch在功能上并无太大差别。
## CyclicBarrier vs CountDownLatch
与CountDownLatch相比，也是他们在功能上似乎是一致的，但是，在实现上，和应用上，有些地方是有很大不同。比如：

- CountDownLatch设置完count数之后，只能使用一次，也就说，count为0之后，这个代码就到头了，不能再次使用
- CyclicBarrier也需要设置初始值，被称"分片"，功能上差不多，也是计数器，但是，如果这个count到0之后，代码会将其进行重置，无需重新初始化一个，可以重复使用。这种特性被称为**循环特性**
- CyclicBarrier分片数不能为0，CountDownLatch可以为0
- CyclicBarrier是有Lock和Condition实现的。CountDownLatch是由AQS

多出的这个特性，就让CyclicBarrier有了很大的发挥空间，可以作为一部分场景解决方案

# 常用方法

- CyclicBarrier(int parties)：设置分片
- CyclicBarrier(int parties, Runnable barrierAction)：设置分片，并且可以在额外设置一个线程，当所有线程在执行到barrier point的时候该线程被调用，用来在任务执行结束之后再执行某些操作
- int getParties()：获取分片数量
- int await()：阻塞当前线程，当所有线程到达障点(barrier point)之后，当分片数为0时，就解除阻塞，进行之后的操作，同时，分片为0时，再次调用会直接返回。返回到达线程的次序数。
- int await(long timeout, TimeUnit unit)：设置超时，如果有线程到达时间超出，那么退出阻塞
- boolean isBroken()：查询此屏障是否处于断开状态
- getNumberWaiting()：返回当前barrier有多少个线程执行了await()方法
- reset()：中断当前barrier，并且重新生成一个Generation的实例，并将计数器重置为初始值

## broken状态
在使用CyclicBarrier时，都会创建一个内部类Generation的实例，该实例只包含一个属性broken，用来标志当屏障是否已经破损，如果破损，就会抛出BrokenBarrierException来终端执行。初始化默认为false。一般有如下情况：

- 如果正在等待的线程中断了，那么broken就会为true
- 如果正在等待的线程超时了，那么broken就会为true
- 如果正在等待的线程被reset()方法重置了，broken为false

# 常见场景

## CountDownLatch类似
使用它基础的特性，作为等待其他线程到达，然后进行主线程操作

{% codeblock lang:java %}

public static void main(String[] args) throws BrokenBarrierException, InterruptedException {

    List<Goods> list = Goods.getGoodsList();
    // 初始化CyclicBarrier，并设置分片，作用与计数器类似。这里多设置一个主线程
    final CyclicBarrier cyclicBarrier = new CyclicBarrier(list.size() + 1);
    list.forEach(goods -> {
        new Thread(() -> {
            try {
                // 模拟业务逻辑
                TimeUnit.MILLISECONDS.sleep(20);
                goods.setThreadContent(Thread.currentThread().getName());
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                try {
                    // 子线程运行结束时，等待其他线程也到达这个点
                    int await = cyclicBarrier.await();
                    System.out.println("到底次序：" + await);
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    });
    // 把主线程也作为一个子线程，等待所有子线程到达
    int await = cyclicBarrier.await();
    System.out.println("到底次序：" + await);
    list.forEach(System.out::println);
}
{% endcodeblock %}

## 循环特性
模拟游客出现的场景，展现循环特性。 模拟用户等车，清点人数，上车，开车，下车，清点人数，停车这个流程。
{% codeblock lang:java %}

public static void main(String[] args) throws BrokenBarrierException, InterruptedException {

    // 11人
    CyclicBarrier barrier = new CyclicBarrier(11);
    for (int i = 0; i < 10; i++) {
        new Thread(new Tourist(i, barrier)).start();
    }
    // 主线程阻塞，等待所有人上车中
    int await = barrier.await();
    System.out.println("All people get on the bus，await：" + await);

    // 主线程阻塞，等待所有人下车中
    barrier.await();
    System.out.println("All people get off the bus");
}

class Tourist implements Runnable {

    private final int personId;
    private final CyclicBarrier barrier;

    public Tourist(int personId, CyclicBarrier barrier) {
        this.personId = personId;
        this.barrier = barrier;
    }

    @Override
    public void run() {
        System.out.printf("People: %d by bus \n", personId);
        // 上车开始
        seconds();
        // 乘客等待上车中
        barrier("Person: %d Get on bus, and wait other people \n");

        System.out.printf("People: %d 行程中 \n", personId);
        // 下车开始
        seconds();
        // 乘客下车等待中
        barrier("People: %d Get off the bus, and wait the other people \n");
    }
    
    // 模式上下车耗时
    private void seconds() {
        try {
            TimeUnit.MILLISECONDS.sleep(20);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void barrier(String message) {
        System.out.printf(message, personId);
        try {
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
{% endcodeblock %}
