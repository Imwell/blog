---
title: Reactor初探  
date: 2021-01-20 23:48:15  
tags:  
    - Reactor
    - 反应式  
categories:
    - Java
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## Reactor
___

在传统的编码中，会将逻辑处理代码写成方法，需要的数据由方法参数传入，处理过的数据由方法的返回值返回。

执行时以main方法为入口点启动，按照一定的顺序执行这些方法，数据依次流入流出每个方法，当所有的方法执行完时，数据也处理完了，就结束了。

整个过程是以逻辑代码的执行为主线，数据只是一个必须的参与者而已，因为代码要处理数据，如果数据不到位，代码就停下来不执行，等待数据的到来。

这就是典型的同步阻塞式的执行过程，非常简单，易于理解，而且代码也很好写。

到目前为止，我们提到的都是响应式的理论，那应该怎样去实现它呢，一时间还真没有头绪。

响应式是异步非阻塞，和同步阻塞应该是相对的。那我们不妨就拿响应式往同步阻塞上套一下，看看能得到什么有价值的发现。

响应式关注两点，变化和反应，而且是变化在前，反应在后。同步阻塞也关注两点，执行逻辑和数据，而且是执行逻辑在前，数据在后。

那就开始建立对应关系。因为“反应”是一系列行为动作，所以应该和“执行逻辑”对应。那“变化”只能和“数据”对应，其实这是对的，“数据”由不可用到可用，本身就是发生了一个“变化”。

这个对应关系建立的很完美，但是逻辑顺序却完全冲突。响应式是由变化主导反应，这很好理解，我都没有变化，你无须做出反应。同步阻塞是由执行逻辑主导数据，这也很好理解，我代码都没执行呢，根本不需要数据。

可见，它们的对应关系非常完美，但主导顺序完全相反，这就是一个非常非常有价值的发现。

因为我们只需把同步阻塞倒过来，就是实现响应式的大致方向。这样的推理貌似是对的，但实际当中是这样的吗？嗯，是这样的。

现在请大家和我一起扭转思维。原来以逻辑代码执行作为主线，数据作为参与者。现在以数据作为主线，逻辑代码执行作为参与者。说的再白一些，**原来是数据传递到逻辑代码里，现在是逻辑代码传递到数据里**。

有人也许会问，逻辑代码怎么传递？哈哈，Lambda表达式呀，函数式编程呀。

想象一下，有一个长长的管子，里面的水一直在流。

如果你想让水变成橙色的，只需在管子上开个口，加装一个可以持续投放橙色染料的装置，结果流经它的水都变成橙色的了。

如果你想让橙色的水变甜的话，只需在后面的管子上开个口，加装一个可以持续投放白糖的装置，结果流经它的水都变成甜的了。

同理，可以在后面继续加装投放柠檬酸的装置，让水变酸，在后面继续加装压入二氧化碳的装置，让水带气泡。

最后发现，自来水经过多道工序处理后变成了芬达。

如果把水流看作是数据流，把投放装置看作是逻辑代码，就变成了，数据先流入第一个逻辑代码，处理后再流入第二个逻辑代码，依次流下去直至结束。

这就是以数据作为主线，逻辑代码只是参与者，同时它也是Reactor实现响应式编程的原理，Spring官方使用的响应式类库就是Reactor。

其中，“以数据为主线”和“在变化时通知处理者”这两个功能Reactor库都已经实现了，我们需要做的就是“对变化做出反应”，即插入逻辑代码。
## Reactor入门
___
在Reactor中，有两个非常重要的类，就是Mono和Flux，它们都是数据源，在它们内部都已经实现了“以数据为主线”和“在变化时通知处理者”这两个功能，而且还提供了方法让我们来插入逻辑代码用于“对变化做出反应”。

Mono表示0个或1个数据，Flux表示0到多个数据。先从简单的Mono开始。

如下有个简单示例：
```
    public static void main(String[] args) {
        displayCurrTime(1);
        displayCurrThreadId(1);
        //创建一个数据源
        Mono.just(10)
                //延迟5秒再发射数据
                .delayElement(Duration.ofSeconds(5))
                //在数据上执行一个转换
                .map(n -> {
                    displayCurrTime(2);
                    displayCurrThreadId(2);
                    displayValue(n);
                    delaySeconds(2);
                    return n + 1;
                })
                //在数据上执行一个过滤
                .filter(n -> {
                    displayCurrTime(3);
                    displayCurrThreadId(3);
                    displayValue(n);
                    delaySeconds(3);
                    return n % 2 == 0;
                })
                //如果数据没了就用默认值
                .defaultIfEmpty(9)
                //订阅一个消费者把数据消费了
                .subscribe(n -> {
                    displayCurrTime(4);
                    displayCurrThreadId(4);
                    displayValue(n);
                    delaySeconds(2);
                    System.out.println(n + " consumed, worker Thread over, exit.");
                });
        displayCurrTime(5);
        displayCurrThreadId(5);
        pause();
    }
    //显示当前时间
    static void displayCurrTime(int point) {
        System.out.println(point + " : " + LocalTime.now());
    }

    //显示当前线程Id
    static void displayCurrThreadId(int point) {
        System.out.println(point + " : " + Thread.currentThread().getId());
    }

    //显示当前的数值
    static void displayValue(int n) {
        System.out.println("input : " + n);
    }

    //延迟若干秒
    static void delaySeconds(int seconds) {
        try {
            TimeUnit.SECONDS.sleep(seconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    //主线程暂停
    static void pause() {
        try {
            System.out.println("main Thread over, paused.");
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
打印结果如下：

```
1 : 23:59:23.948277400
1 : 1
5 : 23:59:23.973340300
5 : 1
main Thread over, paused.
2 : 23:59:28.973939600
2 : 20
input : 10
3 : 23:59:30.998483400
3 : 20
input : 11
4 : 23:59:34.001911800
4 : 20
input : 9
9 consumed, worker Thread over, exit.
```
可以看到不到1秒钟时间主线程就执行完了。然后5秒后数据从数据源发射出来进入第一步处理，2秒后进入第二步处理，3秒后进入第三步处理，数据被消费掉，就结束了。其中主线程Id是1，工作线程Id是9。

这段代码其实是建立了一个数据通道，在通道的指定位置上插入处理逻辑，等待数据到来。

主线程执行的是建立通道的代码，主线程很快执行完，通道就建好了。此时只是一个空的通道，根本就没有数据。

在数据到来时，由工作线程执行每个节点的逻辑代码来处理数据，然后把数据传入下一个节点，如此反复直至结束。

**所以，在写响应式代码的时候，心里一定要默念着，我所做的事情就是建立一条数据通道，在通道上指定的位置插入适合的逻辑处理代码。同时还要切记，主线程执行完时，只是建立了通道，并没有数据。**

**转自**：https://www.cnblogs.com/lixinjie/p/step-into-reactive-programing-in-an-hour.html
