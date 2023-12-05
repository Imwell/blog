---
title: Java-基础（一）
date: 2022-12-07 14:40:21
tags:
  - Java
categories:    
  - Java
description: Java基础(一)
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## 核心点
1. 浮点数精度丢失: 由于浮点数在计算中的存储机制，转换为二进制存储会出现截断导致不精确的问题。[参考](http://kaito-kidd.com/2018/08/08/computer-system-float-point/)
2. 对象中`hashCode()`与`equals()`的作用: 
   1. equals()方法可以用于对比对象是否相同，它是属于Object这个公共父类的方法。如果不进行重写，那么比较的会是变量的地址值。
   2. hashCode()方法主要是用于获取哈希码，主要用在hash表中，也是属于Object类的方法。散列表中才有用，在其它情况下没用。
   3. hashCode()方法的存在，对于一些HashSet等散列结构的集合提供了便利，大大提高了执行效率。因为HashSet类似的集合，在进行插入等操作时，会进行对比数据是否存在，也就是相等，那么，`hashCode()`方法就能直接使用，作为第一步。
   4. 在非散列结构的集合中，对象比较相等时，`hashCode()`方法不会生效
3. 异常的分类:
   1. Exception: 程序本身可以处理的异常，可以通过 catch 来进行捕获
   2. Error: 程序无法处理的错误，不建议使用catch捕获，这些异常发生时，Java 虚拟机（JVM）一般会选择线程终止。
4. 反射:
   1. 什么是反射: 可以在运行时，通过反射技术来获取到目标类及其类的方法，并执行该类的方法。也就是说，这是一种手段，用来在任意情况下获取类及其属性、方法，并执行方法。
   2. 反射的作用: 反射并不是独立的技术，它是融合与其它方法之中的，比如Spring，Mybatis等框架中的动态代理中。
   3. 优点: 代码可以更加灵活，打破静态语言的劣势
   4. 缺点: 安全问题，效率略低
5. SPI: (Service Provider Interface) 服务提供接口
   1. 什么是SPI: 专门提供给服务提供者或者扩展框架功能的开发者去使用的一个接口。和一般API接口相反，上游指定规范，下游实现。
   2. 常见应用: 日志接口，数据库驱动加载
   3. 优点: 解耦了调用方和实现方，调用方只需要调用，不需要配置实现方进行逻辑细节调整，各司其职。
   4. 缺点: 需要遍历加载所有的实现类，不能做到按需加载，这样效率还是相对较低的。当多个`ServiceLoader`同时`load`时，会有并发问题。
6. 序列化: 当数据需要持久化或者在网路传输时，需要将数据进行序列化，也就是将数据转换为二进制流的过程。常见场景序列化比如: 网络传输，DB(例如: redis)操作，File读取写入，内存，cloud。序列化对应TCP/IP四层模型中的应用层
7. 集合: 主要包含两个大类：
   1. Collection集合类，包含三个子类：
      1. Set: 无序集合，不可重复
      2. List: 有序集合，可重复
      3. Queue: 队列，特定顺序存储，有序可重复
   2. Map集合: k-v结构集合，k不可重复且无序。包含：
      1. HashMap: 当链表长度大于阈值(8)，并且数组长度大于64时，会将链表转换为红黑树，提高查询效率，但线程不安全。
      2. Hashtable: 数组+链表组成，线程安全。
      3. TreeMap: 红黑树
      4. LinkedHashMap: 继承自HashMap，大体结构与HashMap主要差别在有一条双向链表，可以保持查询顺序，解决了Map集合k值无序的问题，实现了顺序相关逻辑。
8. 无序性：无序不等于随机，而是指存储的数据在底层数组中并非按照数组索引的顺序添加，而是根据数据的哈希值决定的。
9. Queue，Deque，ArrayDeque，LinkedList，PriorityQueue: 
   1. Queue：Queue是单端队列，FIFO原则
   2. Deque：Deque是双端队列
   3. ArrayDeque：实现了Queue接口，基于可变长的数组和双指针来实现，不支持NULL存储，插入存在扩容可能性，但是O(1)，性能好，用于实现队列比较好。
   4. LinkedList：基于数组和链表实现，一般不用。
   5. PriorityQueue：更多的会出现在手撕算法的时候
10. HashMap和Hashtable区别:
    1. 线程是否安全
    2. 是否能存储null：HashMap支持k,v为null，但null的k只能有一个
    3. 底层数据结构:
    4. 初始容量和扩容大小: Hashtable初始为11，HashMap为16。
    5. 效率：线程安全的效率低
11. HashMap与HashSet的区别: 基本没区别，因为HashSet是通过HashMap实现的
12. HashMap 的长度为什么是 2 的幂次方: 为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀
13. ConcurrentHashMap 线程安全的具体实现方式: 采用 Node + CAS + synchronized 来保证并发安全。数据结构和`HashMap` 1.8的数据结构相似，数组+链表/红黑树。
14. 集合使用中的常见问题:
    1. 集合判空使用`isEmpty()`
    2. 集合转Map注意value为null的情况
    3. 不能在循环过程中对集合进行增删操作
    4. 集合转数组，一定要使用`toArray()`方法
    5. 数组转集合的方法`asList()`，转换之后的集合，不能对其进行增删操作，否则会抛出异常。建议使用其它方法进行转换
15. IO流: 以下为四个基础流 
    1. InputStream：字节输入流，用于从源头（通常是文件）读取数据（字节信息）到内存中，所有字节输入流的父类
    2. OutputStream：字节输出流，用于将数据（字节信息）写入到目的地（通常是文件），所有字节输出流的父类
    3. Read：字符输入流，用于从源头（通常是文件）读取数据（字符信息）到内存中，所有字符输入流的父类，用于读取文本
    4. Writer：字符输出流，用于将数据（字符信息）写入到目的地（通常是文件），所有字符输出流的父类
    5. 为啥有字节流的情况下，还有字符流？
       1. 在不确认编码的情况下，使用字节可能会出现乱码，比如读取中文文件
       2. 面对不同情况下，合理使用字节或者字符流，比如图片等，使用字节流，涉及到字符的情况下，使用字符
16. 字节缓冲流: 为了提高流的传输效率，避免频繁的IO操作，可以使用缓冲流进行代替
    1. BufferedInputStream: 字节缓冲输入流。从源头（通常是文件）读取数据（字节信息）到内存的过程中不会一个字节一个字节的读取，而是会先将读取到的字节存放在缓存区，并从内部缓冲区中单独读取字节。这样大幅减少了 IO 次数，提高了读取效率。默认字节大小8192
    2. BufferedOutputStream: 字节缓冲输出流。
    3. BufferedReader: 字符缓冲输入流
    4. BufferedWriter: 字符缓冲输出流
17. Unix的IO模型: 同步阻塞I/O、同步非阻塞I/O、I/O多路复用、信号驱动I/O、异步I/O
18. Java的IO模型: 
    1. BIO: 同步阻塞I/O模型
    2. NIO: I/O多路复用模型，主要使用这个模型
    3. AIO: 异步I/O模型,基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作
19. 字符编码所占字节数: `utf8`: 英文1字节，中文3字节，`unicode`: 全部为2字节，`gbk`: 英文1字节，中文2字节 
20. ArrayList扩容: 
    - 添加数据
    - 判断内容集合长度size和源数组长度，如果相同则扩容，不相同则新增一个
    - 复制扩容
    - 扩容后的大小: 源数组长度 + 0.5*原数组长度 = 1.5 * 源数组长度
    - 和最大数组长度对比，返回正确的值

```
# java11
public boolean add(E e) {
    // 结构上的修改次数
    modCount++;
    // e: 数据
    // elementData: 数据集合
    // size: 长度
    add(e, elementData, size);
    return true;
}

private void add(E e, Object[] elementData, int s) {
    // 1. 判断长度，如果长度size和数据长度相同，则扩容
    if (s == elementData.length)
        // 2. 扩容
        elementData = grow();
    // 不同则新增在数组中
    elementData[s] = e;
    // 长度加一
    size = s + 1;
}
private Object[] grow() {
    return grow(size + 1);
}
vprivate Object[] grow(int minCapacity) {
    // 3. 复制扩容
    return elementData = Arrays.copyOf(elementData,
                                       newCapacity(minCapacity));
}
// 扩容核心逻辑
private int newCapacity(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 新容积 = 源数组长度 + 右移动1位(实际上就是原数字/2)
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 这个情况适用于数组为空的情况
    if (newCapacity - minCapacity <= 0) {
        // 赋值默认长度10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return minCapacity;
    }
    // 和最大数组长度对比，返回正确的值
    return (newCapacity - MAX_ARRAY_SIZE <= 0)
        ? newCapacity
        : hugeCapacity(minCapacity);
}
```
## 查缺补漏

1. 位移运算符: 
   - 左移(<<): 左移若干位置，高位丢弃，低位补零；`x << 1`相当于 1 * 2；
   - 右移(>>): 右移若干位置，高位补符号位，低位丢弃。正数高位补0，负数补1。
   - 无符号右移(>>>): 无符号右移，忽略符号位，空位都以0补齐
2. 包装类的缓存机制: 包装类型基本上都使用了缓存机制来提升性能。`Byte,Short,Long,Integer`这四种包装类都是默认创建了[-128,127]的相应类型的数据缓存，`Character`创建了[0,127]范围的缓存数据
3. 超出long类型上限的数据怎么存放? 使用`BigInteger`存放，内部是一个`int[]`数组
4. 引用拷贝，浅拷贝，深拷贝
![拷贝分析.png](/img/copy_img.png)

5. Object的方法
   - notify(): 唤醒一个在此对象监视器上等待的线程
   - wait(): native方法，暂停线程的执行，释放锁
   - clone(): 创建并返回当前对象的一份拷贝，是浅拷贝
6. String是不可变的:
   - 保存字符串的数组被`final`修饰并为私有
   - String类被`final`修饰导致不能继承，避免被子类破坏其不可变性
7. 字符串常量池: JVM为了提升性能和减少内存消耗专门针对字符串开辟的一块空间，主要目的是为了避免字符串重复创建
8. String#`intern()`方法: 使用native修饰的本地方法，其作用是用来将其引用保存在常量池中，如果有字符串创建，后续有两种情况
   - 如果通过`equals()`判断位相同，则会从常量池中查询并返回
   - 新创建一个，并保存进常量池，并返回其引用
   
     {% codeblock lang:java %}
     String s1 = "abc";
     String s2 = s1.intern();
     String s3 = new String("abc");
     String s4 = s3.intern();
     // s1创建对象，并存入常量池，s2直接拿取
     System.out.println(s1 == s2); // true
     // s3创建新对象，s4也是从常量池中拿取
     System.out.println(s3 == s4); // false
     // s1和s4都是相同的引用，所以相同
     System.out.println(s1 == s4); // true
     {% endcodeblock %}
   
9. 常量折叠: jvm会把常量计算的值进行求解，并把结果赋值给对应的变量，然后保存到常量池中，比如: `String i = "abc" + "1"`，经过编译优化之后为: `String i = "abc1"`。但也并不是所有常量都支持折叠，引用变量在编译期间不确定具体值，就不能折叠优化
10. try-catch-finally: 场景的异常捕获处理方式，try中放入可能抛出异常的逻辑，捕获异常，catch用来处理异常，finally模块无论是否捕获异常都会执行。**如果没有异常，该模块先于return执行** 
    - finally不执行的情况: 虚拟机被终止，线程死亡，关闭CPU
    - finally中不要执行return。根据上面finally模块的接受，如果该模块存在return，就会有先于try模块执行
    - 字节码角度下该语法糖的实现方式
11. try-with-resources: 在处理一下有需要进行关闭流的操作时，我们可以使用它来替换try-catch-finally。这样我们的代码更加简洁以及获取到的异常信息更加明确
12. Unsafe魔法类: 提供了一些可以直接操作内存、自主管理内存资源的操作，可以提供Java运行效率，但是，过多使用也可能代理不安全的问题，要谨慎。
13. 本地方法(NativeMethod): Java中使用其它语言编写的方法，使用`native`关键字作为修饰符，Unsafe类中的方法都是依赖本地方法
    

### 动态代理
    单纯的静态代理不符合实际需要，我们就需要能够动态的生成代理类，这样就能够在很多场景下进行再操作
#### Java动态代理
    InvocationHandler接口和Proxy类是核心。代理实例上调用方法时，将对代理的方法进行编码，并分配到对应的处理程序的invoke方法

{% codeblock lang:java %}
public interface InvocationHandler {
    // 当你使用代理对象调用方法的时候实际会调用到这个方法
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}

public class Proxy  {
    // 生成一个代理对象
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException
    {
        ......
    }
}
{% endcodeblock %}

#### Cglib代理
Java代理方式，如果不是接口就不能够使用，为了弥补这个缺点，就出现了cglib的代理方式。当然，使用它需要引入新的依赖
```xml
<dependency>
  <groupId>cglib</groupId>
  <artifactId>cglib</artifactId>
  <version>3.3.0</version>
</dependency>
```
它也有一个接口`MethodInterceptor`去实现对应代理类的增强。它需要创建一个增强类`Enhancer`，并设置相对应的属性，比如: 类加载器，被代理类，对应拦截器对象
{% codeblock lang:java %}
// 需要实现该方法拦截器，后续会注入到增强类中，用来实现具体的代理类
public interface MethodInterceptor extends Callback {
    // var1: 代理的对象；var2: 代理的方法，
    Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
}
// 自己定义一个工厂类获取代理。类似Jdk代理的newProxyInstance
public class CglibFactory {
    public static Object getProxy(Object target) {
        Enhancer enhancer = new Enhancer();
        // 类加载器
        enhancer.setClassLoader(target.getClass().getClassLoader());
        // 父类为实例类
        enhancer.setSuperclass(target.getClass());
        // 方法拦截器
        enhancer.setCallback(new CglibProxyHandler());
        // 创建代理类
        return enhancer.create();
    }
}
{% endcodeblock %}

#### 两种代理方式对比
1. jdk只能用来代理实现了接口的类或者接口类，而cglib可以弥补以上缺点，代理类无限制。
2. jdk效率更好

#### 静态代理与动态代理对比
1. 动态代理更加灵活，不需要针对每个目标去实现一个代理类
2. 静态代理在编译时就将其编译为字节码(.class)文件，再解释运行。动态代理是运行时动态生成类字节码，并加载到JVM中
