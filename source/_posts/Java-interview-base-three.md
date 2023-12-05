---
title: Java面试-基础（三）
date: 2023-01-30 18:13:18
tags:
  - Java
categories:
  - Java
description: Java基础(二)
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## JVM垃圾回收

1. GC回收的主要分类:
   1. 部分回收(Minor GC)
      1. 新生代收集: 只对新生代的回收
      2. 老年代收集: 只对老年代进行垃圾回收
      3. 混合收集: 堆整个新生代和部分老年代进行垃圾回收
   2. 全部回收(Full GC): 回收整个Java堆和方法区
2. GC回收的空间分配担保: GC每次回收都会计算老年代连续空间和新生代对象所占内存的大小关系，如果老年代大于等于新生代，那么就会进行一个新生代的回收，否则就会进行全部回收。
3. 死亡对象判断方法: 堆中存放着几乎所有的对象实例，每次回收前都会判断是否死亡。JVM通过如下几种方法进行判断:
   1. 引用计数法: 通过计数器表明引用次数，简单快捷，但是不用这个方法作为判断标准，因为**循环引用**的原因。例如：A和B相互引用，GC就回收不了这两个类的对象实例，因为他们相互引用，计数器不为0
   2. 可达性分析算法: 通过一个`GC Roots`作为起点，通过一个个节点进行向下查找，这个链路叫做**引用链**，如果一个对象找不到`GC Roots`，那么这个对象就没有被引用，就应该被回收
4. 成为`GC Roots`的标准
5. 如果对象实列被确认为废弃对象，能够被回收，这个时候其实只是处在可以被回收阶段，等待回收中。之后要真正的进行回收需要进行两次标记，然后才会被真正的回收。
6. Java常量回收: 方法区的运行时常量池会存储一些常量，那么如何判断常量是否废弃？
   1. 废弃的常量: 如果没有对象引用对应的常量，那么这个常量就是废弃的，需要被垃圾回收
7. 无用的类: 如何判断一个类是属于无用的?必须满足下面三个条件
   1. 该类的实例全部被回收
   2. 加载类的ClassLoader已经被回收
   3. 该类的Class对象没有在任何地方被引用，无法在任何地方通过反射来访问该类

### Java的引用

什么是引用: 
   1. jdk1.2的时候，如果引用类型的数据存储的数值代表的是另一块内存的起始地址，那这块内存代表一个引用
   2. jdk1.2之后，这个概念进行扩展，又分为几大引用
      1. 强引用: 必不可少的引用，大部分都属于该引用，垃圾回收不会回收强引用，抛出oom也不会回收
      2. 弱引用: 和软引用类似，但生命周期更短。只要gc线程发现弱引用，就会回收
      3. 软引用: 处于可有可无状态。内存够，不回收，内存不够，就会回收。用来实现内存铭感的高速缓存
      4. 虚引用: 形同虚设，只有虚引用的对象，等于没有引用。主要用于跟踪垃圾回收的活动
   3. 一般情况下，只有软引用会被用到，可以加速垃圾内存的回收速度，防止内存溢出等问题
   
## 垃圾回收算法
1. 标记-清除算法: 标记不需要回收的对象，然后去回收没有标记出来的对象。**最基础**的算法，后续都是对其进行补足
2. 标记-复制算法: 原理和上面的算法相同，是为了补足其**效率问题**。把内存分为大小相同的两块，每次都是使用其中一块，然后根据标记把不需要回收的对象复制到另一块没用过的内存，再清除使用过的那块，这样效率更高，但是内存使用更多。
3. 标记-整理算法: 原理和第一个算法相同，但是标记完，我们会让其标记过的内存块向一端移动，然后清理超出边界的内存区域
4. 分代收集算法: 当前虚拟机采用的垃圾回收算法，根据堆的新生还是老年代对其选择合适的上述算法进行垃圾回收。
5. 举个例子: 新生代中，对象的创建和销毁都很频繁，那么就需要提高效率，使用`标记-复制`算法就很合适，老年代中销毁不是很频繁，但是需要考虑到空间问题，那么使用`内存-整理`或者`内存-清除`就比较好。**同时，这也是jvm堆区分老年代和新生代的原因之一**

## 常见的垃圾回收器及其算法

1. Serial: 古老，单线程收集器。标记-复制，标记-整理算法
2. ParNew: Serial的多线程版本。多线程进行垃圾回收，其它和Serial一致。新生代采用`标记-复制算法`，老年代采用`标记-整理算法` 
3. Parallel Scavenge: `标记-复制算法`的多线程垃圾回收器，更多关注的是吞吐量(cpu中用于运行用户代码的时间与cpu总消耗时间的比值)，高效利用cpu。
4. Serial Old: Serial的老年代收集器版本，单线程，jdk1.5及其之前版本配合Parallel收集器使用 
5. Parallel Old: Parallel Scavenge收集器老年代版本。使用多线程和标记-整理算法。注重吞吐量和cpu资源的场景使用
6. CMS(Concurrent Mark Sweep): 获取最短回收停顿为目标的收集器。符合非常注重用户体验的应用。是HotSpot虚拟机第一款真正意义上的并发收集器。实现了垃圾回收和用户线程同时工作。收集分为四个过程:
   1. 初始标记: 暂停所有线程，标记所有与GC Roots相连的对象，速度很快
   2. 并发标记: 同时开启gc与用户线程，跟踪记录可达对象引用的更新
   3. 重新标记: 修正并发标记里发生变动的对象，
   4. 并发清除: 开启用户线程，同时gc对为标记的区域做回收
![cms](/img/CMS-gc.jpeg)
- 但同时也有三个缺点:
   - 对cpu资源铭感
   - 无法处理浮动垃圾
   - 会有大量空间碎片
7. G1(Garbage first): 面向服务器的垃圾收集器。针对多核心cpu和大内存的机器。既能满足gc停顿需求，又具备高吞吐量。拥有如下特点:
   1. 并行与并发: 多核心cpu能够充分利用。g1通过并发让gc动作在不停顿线程的情况下继续运行
   2. 分代收集: 不同代的gc处理
   3. 空间整理: 整体上是使用标记-整理算法实现，但是局部是使用标记-复制算法
   4. 可预测的停顿: 可以预测停顿时间模型，明确指定一个长度m毫秒的时间的内进行停顿gc
- 运行步骤:
    - 初始标记
    - 并发标记
    - 最终筛选
    - 筛选回收
- g1回收器在后台维护了一个列表，每次根据运行的回收时间，会优先选择价值最大的region。可以保证最大效率的执行回收。
8. ZGC: 采用标记-复制算法，并对其进行了重大改进

## 类文件结构解析
Class文件通过ClassFile定义
```text
ClassFile {
    u4             magic; //Class 文件的标志
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
    u2             constant_pool_count;//常量池的数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//Class 的访问标记
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个类可以实现多个接口
    u2             fields_count;//Class 文件的字段属性
    field_info     fields[fields_count];//一个类可以有多个字段
    u2             methods_count;//Class 文件的方法数量
    method_info    methods[methods_count];//一个类可以有个多个方法
    u2             attributes_count;//此类的属性表中的属性数
    attribute_info attributes[attributes_count];//属性表集合
}
```
1. 魔数(Magic Number): 确定文件是否为一个能被虚拟机接收的Class文件
2. Class文件版本号: 分为两个版本号，次版本和主版本
3. 常量池(Constant Pool): 分别记录常量池数量和常量池。包含字面量和符号引用
4. 访问标志: 标识类或者接口层次的访问信息
5. 字段表合集(Field): 用于描述接口或类中声明的变量，不包括局部变量。
6. 方法表集合(Method): 包括方法数量和方法具体信息。
7. 属性表集合(Attributes): 方法表和字段表都会拥有自己的属性表集合。

字段表和方法表都有相同的数据结构:
```text
field_info {
    u2 access_flags; // 字段作用域
    u2 name_index; // 字段名
    u2 descriptor_index; // 字段和方法的描述符
    u2 attributes_count; // 额外属性
    attribute_info attributes[attributes_count]; // 具体属性具体内容s
}
```

## 类加载过程

Class文件要加载到虚拟机中才能运行，那么如何加载呢？有如下几步:
1. 加载: 完成类文件的加载，有一个方法区的访问入口
2. 链接: 这个过程又分为几步:
   1. 验证:
      - 文件格式: 是否符合类文件格式
      - 元数据: 对字节码描述的信息进行语义分析事实，保证描述的信息符合java语言规范要求
      - 字节码: 最复杂，但也很终于，需要校验程序语言的合法以及逻辑问题
      - 符号引用: 确保能够正确执行解析动作
   2. 准备: 准备为对象分配内存并进行初始化的阶段，这个分配是在方法区，那么分配就是所属类的变量，实例对象是在堆中分配内存
   3. 解析: 虚拟机将常量池内的符合引用替换为直接引用的过程。解析主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用限定符7类符号引用
3. 初始化: 正式执行类中定义的Java程序代码。五种情况下，类必须初始化
   1. 遇到`new`，`getststic`，`putstatic`，`invokestatic`这4条直接码指令时
   2. 使用反射包方法对类进行反射调用时
   3. 初始化一个类，如果父类还没初始化，优先初始化父类
   4. 虚拟机启动时，用户需要定义一个启动类，比如`main`方法
   5. MethodHandle和VarHandle可以看作是轻量级的反射调用机制，使用它们需要先使用findStaticVarHandle来初始化要调用的类
   6. 一个接口中定义了jdk8之后加入的默认方法(被default关键词修饰的接口方法)时，如果接口的实现了发生了初始化，那该接口要在之前被初始化
4. 卸载: class对象被gc。卸载类需要满足三个条件:
   1. 该类的所有对象实例都被gc了
   2. 该类没有其它地方的引用
   3. 该类的类加载器的实例已经被gc

## 类加载
一个非组数类的加载阶段是可控性最强的，我们可以自定义类加载器去控制字节流的获取方式。数组类型不通过类加载器，是jvm直接创建。

**常见的类加载器**:
1. BoostrapClassLoader: 启动类加载器
2. ExtensionClassLoader: 扩展类加载器
3. AppClassLoader: 应用程序类加载器，加载classpath下的所有jar包和类

## 双亲委派模式 
1. 什么是双亲委派: 在类加载的时候，会先判断是否被加载过。被加载过直接返回，没有则尝试加载。加载的时候，会先委派给父类的`classload()`处理，因此所有的
请求都会传送到顶层的类加载器`BoostrapClassLoader`中。当父类无法加载时，才会向下寻找并尝试使用别的类加载器，或又自己加载。当父类加载器为null，会启用
`BoostrapClassLoader`作为父类加载器。
![classloader](/img/classloader.png)
2. 优势: 保证Java程序稳定加载，避免类的重复加载，保证Java的核心API不被篡改。
3. 自定义类加载器: 除了启动类加载器，其它的自定义加载器都需要继承`java.lang.ClassLoader`。

## JVM的核心参数
1. 指定堆内存
   1. -Xms: 堆最小内存
   2. -Xmx: 堆最大内存
2. 指定新生代内存
   1. -XX:NewSize: 新生代内存
   2. -XX:MaxNewSize: 最大新生代内存
   3. -Xmn: 新生代内存，与上面3，4相同
   4. -XX:NewRatio: 配置老年代和新生代内存的比值
3. 指定永久代和元空间
   1. -XX:PermSize: 方法区内存初始大小
   2. -XX:MaxPermSize: 方法区最大内存
   3. -XX:MetaspaceSize: 元空间初始大小
   4. -XX:MaxMetaspaceSize:  元空间最大的大小
4. gc配置: 
   1. 可以根据不同的场景选择不同的垃圾回收算法
   2. 打印gc日志
5. OOM问题: 可以将一些内存错误转移到别的文件里，进行查询错误原因
   1. -XX:+HeapDumpOnOutOfMemoryError: 将遇到的内存溢出错误是将heap转储到物理文件中
   2. -XX:HeapDumpPath=./java_pid<pid>.hprof: 转储写入的文件路径及其文件名
   3. -XX:OnOutOfMemoryError="< cmd args >;< cmd args >": 内存不足时紧急发出的命令，例如`-XX:OnOutOfMemoryError="shutdown -r`命令，可以重启服务器
   4. -XX:+UseGCOverheadLimit: 它限制在抛出 OutOfMemory 错误之前在 GC 中花费的 VM 时间的比例
6. 其它
   1. -server: 启用"Server HotSpot VM"
   2. -XX:SurvivorRatio: eden/survivor 空间的比例, 例如-XX:SurvivorRatio=6 设置每个 survivor 和 eden 之间的比例为 1:6
   3. -XX:MaxHeapFreeRatio : 设置 GC 后, 堆空闲的最大百分比，以避免收缩。
7. **gc优化策略**: 由于全部回收的成文高于部分回收，那么就应该让新对象基本都留在新生代，避免进入老年代。可以根据gc日志分析新生代空间配置大小是否合理，从而使用`-Xmn=`命令调整新生代内存大小，最大限度降低新对象进入老年代的情况。
8. 常用jdk命令 [监视与管理控制台](https://javaguide.cn/java/jvm/jdk-monitoring-and-troubleshooting-tools.html#jconsole-java-监视与管理控制台)
   - jps (JVM Process Status）: 类似 UNIX 的 ps 命令。用于查看所有 Java 进程的启动类、传入参数和 Java 虚拟机参数等信息；
   - jstat（JVM Statistics Monitoring Tool）: 用于收集 HotSpot 虚拟机各方面的运行数据;
   - jinfo (Configuration Info for Java) : Configuration Info for Java,显示虚拟机配置信息;
   - jmap (Memory Map for Java) : 生成堆转储快照;
   - jhat (JVM Heap Dump Browser) : 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果;
   - jstack (Stack Trace for Java) : 生成虚拟机当前时刻的线程快照，线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合
9. jdk可视化工具
   - JConsole
   - Visual VM

```text
# gc日志打印参数
# 必选
# 打印基本 GC 信息
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
# 打印对象分布
-XX:+PrintTenuringDistribution
# 打印堆数据
-XX:+PrintHeapAtGC
# 打印Reference处理信息
# 强引用/弱引用/软引用/虚引用/finalize 相关的方法
-XX:+PrintReferenceGC
# 打印STW时间
-XX:+PrintGCApplicationStoppedTime

# 可选
# 打印safepoint信息，进入 STW 阶段之前，需要要找到一个合适的 safepoint
-XX:+PrintSafepointStatistics
-XX:PrintSafepointStatisticsCount=1

# GC日志输出的文件路径
-Xloggc:/path/to/gc-%t.log
# 开启日志文件分割
-XX:+UseGCLogFileRotation
# 最多分割几个文件，超过之后从头文件开始写
-XX:NumberOfGCLogFiles=14
# 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=50M
```
