---
title: Java并发包之原子类型
date: 2021-02-01 15:17:46
tags:
    - Java
    - 并发
categories:
    - 并发
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## 基础知识
Java内存模型的三个特性：
- **有序性**：线程操作的有序性
- **原子性**：某个操作执行的时候，整个过程要么成功，要么失败，不允许因为某些原因中断而导致部分失败或成功
- **可见性**：线程之间的可见性，一个线程修改之后，另外一个线程也能可见
  
**volatile**关键字：只要被这个关键字修饰，从Java内存模型上来说，那么就代表该变量具有了有序性和可见性，但是还是不能保证原子性。在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制。当对非volatile变量进行读取时，一般都是从内存拷贝到CPU缓存，volatile修饰的变量，则是直接从缓存中拿取。

## AtomicInteger
非原子性操作：int类型的自增或者自减
{% codeblock lang:java %}
/**
 * 这个过程不是原子性的，会涉及到线程安全问题，如果要解决这个问题就需要通过synchronized关键字或者加锁才行，
 * 1.5版本之后，就出现了AtomicInteger等专门用来解决这个问题的工具集
 */
int i = 1;
i++;
i--;
{% endcodeblock %}
AtomicInteger是继承自Number类的子类，它提供了很多原子性操作，主要用户原子递增计数器之类的应用中，但并不能代替整数使用。
{% codeblock lang:java %}
// 源码中的三个关键常量
// Unsafe类
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
// 通过C/C++的本地方法获取value的内存值
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
// AtomicInteger中的变量操作都是针对这个值进行操作的
private volatile int value;
{% endcodeblock %}
   
### AtomicInteger的初始化
{% codeblock lang:java %}
// 传入变量初始化
public AtomicInteger(int initialValue) {
    value = initialValue;
}
// 初始化默认值为0
public AtomicInteger() {
}
{% endcodeblock %}

### AtomicInteger常见操作

{% codeblock lang:java %}
// 获取当前的内存值
public final int get() {
    return value;
}
// 为value设置一个新值
public final void set(int newValue) {
    value = newValue;
}
// 设置值的时候优化内存屏障问题
public final void lazySet(int newValue) {
    U.putIntRelease(this, VALUE, newValue);
}
// 原子性更新操作。经典cas算法，第一个参数传入原本的值，第二个传入新值，通过对比传入原值与AtomicInteger的value是否相等，
// 相等的情况下修改新值成功，否则失败。
public final boolean compareAndSet(int expectedValue, int newValue) {
    return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
}
// 获取原值并自增一，原子性增加操作
public final int getAndIncrement() {
    return U.getAndAddInt(this, VALUE, 1);
}
// 获取原值并自减一，原子性减少操作
public final int getAndDecrement() {
    return U.getAndAddInt(this, VALUE, -1);
}
// 获取自增一之后的值，原子性增加操作
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}
// 获取自减一之后的值，原子性减少操作
public final int decrementAndGet() {
    return U.getAndAddInt(this, VALUE, -1) - 1;
}
// 原子性更新value，返回更新前的值，更新后的值为value+delta的和
public final int getAndAdd(int delta) {
    return U.getAndAddInt(this, VALUE, delta);
}
// 原子性更新value，返回更新后的值，更新后的值为value+delta的和
public final int addAndGet(int delta) {
    return U.getAndAddInt(this, VALUE, delta) + delta;
}
// 以下都是函数式接口
// 获取更新之前的值
int getAndUpdate(IntUnaryOperator updateFunction);
int updateAndGet(IntUnaryOperator updateFunction);
int getAndAccumulate(int x, IntBinaryOperator accumulatorFunction);
int accumulateAndGet(int x, IntBinaryOperator accumulatorFunction);
{% endcodeblock %}

### AtomicInteger的CAS算法
CAS包含三个操作数：内存值Value，旧的预期值expectedValue，要修改的新值newValue。当且仅当内存值与旧的预期值相同时，将内存值修改新值newValue，否则不变。CAS是非阻塞操作。

这种方式也叫乐观锁，如果要对数据进行修改，就需要相对其进行比较，不然在多线程下会出现当前值与内存值不同的情况。

### 自旋方法
CAS算法固然简单方便，但是并不能保证一定会更新成功，如果有必须更新成功的需求，那么就需要在CAS之上进行改进，就出现了自选方法。如下是自旋方法方法getAndAdd的源码
{% codeblock lang:java %}
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        // 通过偏向值获取内存中的Value
        v = getIntVolatile(o, offset);
        // 以更新是否成功为循环条件，更新成功就退出循环，保证能有一次更新成功
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
{% endcodeblock %}

## AtomicBoolean && AtomicLong
**AtomicBoolean**提供了一种原子性读写布尔类型变量的方案，通常用于原子性的更新一个标识，内部实现上value是一个int值，方法较少，和AtomicInteger中的类似。通常AtomicBoolean都会用在一些开关控制操作上。
{% codeblock lang:java %}
// 内部实际的value值为int类型
private volatile int value;

public AtomicBoolean() {
}
// 有参构造为传入一个boolean类型，并更新value为1或0
public AtomicBoolean(boolean initialValue) {
    value = initialValue ? 1 : 0;
}
{% endcodeblock %}

**AtomicLong**也是继承Number的子类，大部分方法和AtomicInteger使用习惯上一致。如果当前机器或者JVM不支持8字节码数字的Lock Free的CAS操作，那么它就会通过synchronized实现锁机制。

### TryLock锁的实现
在使用synchronized关键字加锁时，并不能正确获取锁失败的情况，执行线程只能通过其他线程释放锁才行继续进行。通关使用AtomicBoolean，可以设计一种线程获取锁失败并立即返回的解决方案。有这种机制的锁有显示锁Lock，ReentrantLock，StampedLock。
**注：**书上这个例子有误，需要排查问题
## AtomicReference
**AtomicReference**提供了对对象的原子性读写操作，同时在某些场景下可以替代synchronized和显示锁，实现多线程的非阻塞操作。底层实际上和AtomicInteger一样是CAS函数

应用场景：多线程情况下存钱的非阻塞实现

## AtomicStampedReference
**CAS算法的ABA问题**：多线程情况下，线程T1将值从1修改为2，线程2又把值从2修改为1，但是，这个1已经不是之前的1了，如果线程T3进来，就可以通过原值1，将其修改为3，中间的2可能就会丢失，以2为原值的操作也会丢失。
**AtomicStampedReference**：针对AtomicReference的ABA的解决方案。这个类在初始化的时候会有一个额外的int参数，如同版本号一样，用户自己维护这个版本号，就可以避免ABA问题出现。
{% codeblock lang:java %}
private volatile Pair<V> pair;

// 初始化
public AtomicStampedReference(V initialRef, int initialStamp) {
    pair = Pair.of(initialRef, initialStamp);
}

/**
 * 有个内部类会将两个初始化的变量封装为一个Pair对象
 */
private static class Pair<T> {
    final T reference;
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}
{% endcodeblock %}
   
## AtomicFieldUpdater
通过使用CAS算法提供的Lock Free的乐观方式和synchronized悲观方式，我们都可以实现对共享数据的同步解决方法。除了这两种以外，Java的原子包还提供了一种原子性操作对象的方法：
- AtomicIntegerFieldUpdater：原子性更新对象的int类型，该属性无需声明为AtomicInteger类型
- AtomicLongUpdater：原子性更新对象的Long类型，该属性无需声明为AtomicLong类型
- AtomicReferenceUpdater<T>：原子性更新对象的引用类型属性，无需声明为AtomicReference类型

相较于使用原子类型更加能够节省应用程序内存，但是限制也多，需要注意的地方如下：
- 未被volatile关键字修饰的成员属性无法被原子性更新
- 类变量(静态变量)无法被原子性更新
- 无法直接访问的不能被原子性更新
- final修饰的无法被原子性更新
- 父类的成员属性无法被原子性更新
  
{% codeblock lang:java %}
public static void main(String[] args) {

    AtomicIntegerFieldUpdater<Alex> salary = 
        AtomicIntegerFieldUpdater.newUpdater(Alex.class, "salary");
    Alex alex = new Alex();
    if (salary.compareAndSet(alex, 0, 1)) { // 更新成功
        System.out.println("更新alex成功"); 
        System.out.println(alex);
    } else {
        System.out.println("更新alex失败");
    }

}

class Alex {

    volatile int salary;

    public int getSalary() {
        return salary;
    }
    @Override
    public String toString() {
        return "Alex{" +
                "salary=" + salary +
                '}';
    }
}

{% endcodeblock %}

## Unsafe
Unsafe类是Java提供的对内存进行直接操作，甚至是通过汇编指令对CPU进行操作的工具类。可以用，但需要谨慎使用。Unsafe类正常情况下是不能使用的，因为它不在系统加载器中加载时不能使用的。

Unsafe可以实现的功能：
- 绕过类的构造函数完成对象的创建
- 直接修改内存数据
- 类的加载

## 总结
原子类型包为我们提供了一直无锁的原子性操作共享数据的方法，它们几乎都是通过CAS算法+volatile实现的。
**三个核心点**：
- **volatile**：volatile关键字保证了线程之间的可见性，某线程对volatile修饰的编程做操作时，其他线程也能看到
- **cas算法**：对比交换算法。非阻塞实现同步的核心。由UNSAFE提供，实际上是操作CPU执行来得到保证的。快速失败的方式，在修改已被改变的值的时候会失败。
- **自旋算法**：cas算法对共享数据操作失败时，有自旋算法加持，我们对共享数据的操作最终会得到更新。

无锁可以减少线程阻塞，减少CPU上下文切换，提供程序运行效率。

但是在单线程情况下，synchronized关键字的效率却又高很多，因为在运行期间，JVM对程序进行优化时可以将同步擦除，而原子类是本地方法和汇编指令提供的，Java程序运行期间没有办法优化。


