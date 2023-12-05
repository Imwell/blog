---
title: Java高并发系列(一)--初识线程
date: 2021-03-23 17:48:09
tags:
    - Java
    - 多线程
categories:
    - 多线程
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## 线程
每个任务就是一个进程，线程又是一个进程的基础，每个进程都会至少有一个线程在运行。线程又称为微进程。进程都有自己独立的空间，线程又拥有自己的局部空间及生命周期。
## Java线程的生命周期

线程一般拥有5个主要的生命周期：
- NEW：线程的刚创建的状态，如果没有start，那么这个线程实际还是不存在的。
- RUNNABLE：调用Thread的start方法之后，就进入了RUNNABLE状态，真正意思上的在**JVM进程中**创建了一个线程。但是，它并不会立即执行，而是处在等待CPU调度。等待状态的线程，它只会进入RUNNING状态或者意外中断进程。
- RUNNING：CPU通过轮询或者其他方式获取了执行当前线程的执行权限，真正的开始执行自己的代码。一般，根据不同的情况，它会进入不同的状态。
- BLOCKED：当处于运行状态的线程失去所占用资源之后，便进入阻塞状态。
- TERMINATED：它是一个线程的最终状态，意味着整个生命周期的结束。线程正常结束，线程运行意外或者JVM Crash都会让线程结束

## 线程的创建与启动

{% codeblock lang:java %}

    // 简单创建一个线程
    public static void main(String[] args) { // 通过lamda表达式创建线程
        new Thread(() -> {
            System.out.println("子线程启动");
            sleep();
        }).start();
        System.out.println("主线程");
    }

    new Thread(){ // 通过匿名内部类创建线程
        @Override
        public void run() {
            System.out.println("子线程启动");
        }
    }.start();
    System.out.println("主线程");

{% endcodeblock %}

通过上面代码，我们创建了一个简单的线程，同样，也可以使用匿名内部类的方式创建。可以看到，我们是需要重写一个`run()`方法的，并调用Thread对象的start()方法启动线程。为了了解start()我们可以观察其源码，如下

{% codeblock lang:java %}
    
    public synchronized void start() {
        // 1.状态0表示新创建的线程(NEW)，如果多次启动同一个线程，那么就会抛出异常
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        // 2.把当前线程放入一个group中,默认和父类线程一个组
        group.add(this);

        boolean started = false;
        try {
            start0(); // 4. 线程启动的核心
            started = true;
        } finally {
            try {
                if (!started) { // 3.如果没有正常运行start0()方法，那么就会抛出异常
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native void start0(); // 在jvm中创建线程，实际调用run()方法的地方

{% endcodeblock %}

## Thread与Runnable
通过上面的例子我们了解了基本的创建线程和启动的方式，其中，有需要实现一个匿名内部类的run()的过程，这个run()方法并不是Thread类自己本来就有的，它是实现自Runnable接口的。

**Thread类是创建并启动线程的唯一类**,它实现了Runnable接口，并重写了run()方法。

Runnable接口只有一个run()方法，它是**线程内逻辑执行单元**，如果需要启动线程，并作什么事，就需要子类来实现这个方法，就如同上面例子一样。

一般我们说的创建线程是有两种方式，一种是通过继承Thread类，实现run()的方式。另一种是实现Runnable类的run()方法，然后作为Thread创建时的参数传入的方法。

其实，两种方式的说法是不严谨的，真正创建线程的只有Thread的方式，Runnable的方式，它只是实现执行单元了，创建线程的还是Thread。**综上所述，创建线程的方式只有一种，通过Thread，但是，实现执行单元的方式有两种，继承Thread和实现Runnable。**

我们可以来看看run()方法的源码，它调用的时候就有两种情况。
{% codeblock lang:java %}
    
    @Override
    public void run() {
        // 1.创建时传入Runnable对象，就执行它的run()方法
        if (target != null) {
            target.run();
        }
        // 2.否则，就需要自己实现Thread的run()方法
    }

{% endcodeblock %}

## Thread中的策略模式
策略（Strategy）模式的定义：该模式定义了一系列算法，并将每个算法封装起来，使它们可以相互替换，且算法的变化不会影响使用算法的客户。
策略模式属于对象行为模式，它通过对算法进行封装，把使用算法的责任和算法的实现分割开来，并委派给不同的对象对这些算法进行管理。

通过对策略模式的了解，我们可以发现，Runnable接口类就是将算法进行封装，Thread不需要关心内部如何实现。**Thread只是负责创建和启动线程，Runnable负责算法逻辑**。不光如此，如果使用Runnable一个对象，就可以使用一个执行单元创建多个线程，也就是说，它是作为共享的对象。

## 线程的父子关系

在JDK1.8的Thread的源码中，每个构造方法都会调用init()方法来实例化一个Thread对象，通过源码发现，都会通过`currentThread()`方法来获取当前线程，**一个被创建的线程，都会由另一个线程来创建**。

通过分析线程的生命周期，在线程没有调用start()方法之前，它只能算是一个Thread的实列，并不算是一个线程，那么，`currentThread()`肯定是创建它的那个**父线程**。

{% codeblock lang:java %}

    // JNI方法，存在于JVM本地方法区
    public static native Thread currentThread();

    // 部分源码
    private void init(ThreadGroup g, Runnable target, String name,
            long stackSize, AccessControlContext acc,
            boolean inheritThreadLocals) {

        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();// 获取当前线程作为父线程
        SecurityManager security = System.getSecurityManager()
    // 剩余省略。。。

{% endcodeblock %}

## Thread与ThreadGroup

继续观察init()方法，构造方法第一个参数指定了ThreadGroup，但是，如果没有指定，那么g也会在接下来的流程被赋值。

### SecurityManager
JAVA安全管理器SecurityManager，当运行未知的Java程序的时候，该程序可能有恶意代码（删除系统文件、重启系统等），为了防止运行恶意代码对系统产生影响，需要对运行的代码的权限进行控制，这时候就要启用Java安全管理器。可以通过参数方式启动(可以指定配置文件，不写就不指定)

{% codeblock lang:java %}

    //  获取Java安全管理器
    SecurityManager security = System.getSecurityManager();
    // 如果没有指定ThreadGroup的情况
    if (g == null) {
        /* Determine if it's an applet or not */

        /* If there is a security manager, ask the security manager
           what to do. */
        // 如果管理器存在，则从管理器中获取父类线程，通过分析下面的源码，它也是获取当前线程的线程组
        if (security != null) {
            g = security.getThreadGroup();
        }

        /* If the security doesn't have a strong opinion of the matter
           use the parent thread group. */
        if (g == null) {
            // 获取父类线程的线程组，理论上来说，和通过安全管理器获取的是一样的，因为获取方式和SecurityManager中的一致
            g = parent.getThreadGroup();
        }
    }
    
    // SecurityManager中的getThreadGroup()方法
    public ThreadGroup getThreadGroup() {
        return Thread.currentThread().getThreadGroup();
    }

{% endcodeblock %}

## Thread与JVM 

## Thread常见Api

`sleep(long millis)`：将当前线程睡眠n毫秒
`yield()`：切换当前线程状态，从RUNNING变为RUNNABLE
`setPriority(int new Priority)`：设置线程的优先级。
> 注：
> 1.优先级并不能作为分配线程工作量的指标，它只会在进行线程切换时有更多可能切换到优先级高的线程。
> 2.设置的优先级也有界限，最大为10，最小为1
> 3.给一个子线程设置优先级时，最大优先级不能超过线程组的优先级

`getId()`：获取线程id，线程id在JVM中唯一，从0递增
`currentThread()`：获取当前执行线程的引用
### 线程Interrupt
当一个线程处于阻塞状态时，我们可以使用当前方法的interrupt方法，使其从阻塞中恢复。但是，能恢复阻塞的方法只能是可中断的方法，同时线程也不是死线程。interrupt的主要方法如下：
`void interrupt()`：设置当前线程阻断
`static boolean interrupted()`：
`boolean isInterrupted()`：获取线程是否阻断

打断一个阻塞方法并不等于结束当前线程的生命周期，一旦一个方法在阻塞状态下被打断，都会抛出一个InterruptedException异常。

### 线程Join
`final void join()`
`final synchronzied void join(long millis, int nanos)`
`final synchronzied void join(long millis)`
