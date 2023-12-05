---
title: ByteBuffer使用
date: 2023-07-12 09:10:00
tags:
  - Java
categories:
  - Java
description: 网络传输载体
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---

# ByteBuffer

为了避免操作一下字节数组的繁琐和避免重复造轮子，jdk推出了bytebuffer作为载体操作字节数组。netty是一个网络框架，也推出了类似的载体netty.ByteBuf

## 前置知识

已知NIO的主要包括三个组件: **Buffer**, **Channel**, **Selector**。Buffer就是作为一个消息的载体，在Channel中进行传递，用户可以从中获取消息，也可以把消息放入其中进行传递。

ByteBuffer就是Buffer的子类，是字节缓冲区，特点如下:
- 大小不可变
- 读写灵活
- 支持堆上内存分配和直接内存分配

## Buffer
    缓冲区是特定的基本类型元素的线性有限序列。出去内容，和包含三个重要的元素: capacity, limit, position

- capacity: 元素集包含的数量。它不会为负数也不会变
- limit: 不能读写的第一个元素的索引。它不会为负数也不会比capacity大
- position: 下一个读写的元素的索引。它不会为负数也不会比limit大

### 线程安全

在多线程环境下，buffer不是线程安全的。如果多线程访问，需要进行同步控制

### 只读情况
1. 每个buffer都是可读的，但不是每个都可写。
2. 突变方法在只读缓冲区进行操作，就会抛出 ReadOnlyBufferException 异常
3. 一个只读的缓冲区，内容不可变，但是`position`、`limit`、`mark`是可变的
4. 缓冲区是可以只读，可以通过调用方法`isReadOnly()`来确认

### Marking and resetting

缓冲区的`mark`是在调用reset方法时`position`被重置到的索引。
`mark`不会一定被定义，如果定义它的话，一定不为负数并且小于`position`。
如果`position`或者`limit`被调整为小于`mark`，那么它将被舍弃。
如果`mark`没有定义，调用reset()方法就会抛出异常`InvalidMarkException`

### 大小关系

> 0 <= mark <= position <= limit <= capacity 

一个新创建的buffer通常`position`都为0，`mark`也是未定义。初始化的`limit`也许是0，也许是取决于缓冲区及其构造方法。当然，新分配的缓冲区元素为0

### 常用方法

- clear(): 设置`limit`的值为`capacity`，把`position`置0。把buffer的通道置为**read状态**
- flip(): 设置`limit`的值为当前的`position`，之后再把`position`置为0。把buffer的通道置为**write状态**
- rewind(): 对已经操作的区域可以重新操作，它保留限制不变，并将`position`设置为零
- slice(): 创建一个buffer的子序列。保留`limit`和`position`不变
- duplicate(): 浅拷贝，保留`limit`和`position`不变

## ByteBuffer

相对于buffer，多了三个新的属性:
- byte[] hb: 字节数组。仅仅在heap buffers中用到
- int offset: 偏移量。仅仅在heap buffers中用到
- isReadOnly: 是否只读。仅仅在heap buffers中用到

### 创建ByteBuffer

| 方法                                         | 说明                                                           | 类型              |
|:-------------------------------------------|:-------------------------------------------------------------|:----------------|
| allocateDirect(int capacity)               | capacity 和 limit 相同。position 为0。mark 为 -1。hb 为null           | DirectByteBuffer |
| allocate(int capacity)                     | capacity 和 limit 相同。position 为0。mark 为 -1。hb 为cap长度的byte[]   | HeapByteBuffer  |
| wrap(byte[] array,int offset, int length)  | hb为array。limit为array + offset 的和。cap为array的长度。potision为off的值 | HeapByteBuffer  |
| wrap(byte[] array)                         | hb为array。offset 为0。limit与cap均为array的长度。                      | HeapByteBuffer  |

因为ByteBuffer是抽象类，所以基本都是使用实现类实例化，它主要有两个实现类:
- HeapByteBuffer: 在堆上分配内存
- DirectByteBuffer: 在直接内存中分配内存

{% codeblock lang:java %}

DirectByteBuffer(int cap) {                   // package-private
    // 父类初始化
    super(-1, 0, cap, cap, null);
    // 
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 创建给定大小的直接内存块，会有内存溢出问题
        base = UNSAFE.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    UNSAFE.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 它所关联的这个对象被jvm回收时就会触发 Cleaner.clean 方法，可以及时的释放堆外内存空间
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}

// clean方法
public void clean() {
    if (remove(this)) {
        try {
            // 执行Deallocator的run()方法
            this.thunk.run();
        } catch (final Throwable var2) {
            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                public Void run() {
                    if (System.err != null) {
                        (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                    }
                    System.exit(1);
                    return null;
                }
            });
        }
    }
}

// run()方法
public void run() {
    if (address == 0) {
        // Paranoia
        return;
    }
    // 释放从allocateMemory处获得的内存块
    UNSAFE.freeMemory(address);
    address = 0;
    Bits.unreserveMemory(size, capacity);
}

{% endcodeblock %}

### Buffer中的部分方法

以下方法都会出现在其实现类中，比如获取position等参数

{% codeblock lang:java %}
// 变量
// 对于堆字节缓冲区，此字段将是相对于数组基址和该数组的偏移量的地址
// 对于直接缓冲区，它是内存区域的起始地址
long address;

// 判断当前position是否大于limit，如果小于的话，position自增1
final int nextPutIndex() {                          // package-private
    int p = position;
    if (p >= limit)
        throw new BufferOverflowException();
    position = p + 1;
    return p;
}
// 判断limit和position
final int nextPutIndex(int nb) {                    // package-private
    int p = position;
    if (limit - p < nb)
        throw new BufferOverflowException();
    position = p + nb;
    return p;
}
// 计算offset之后的值
final int checkIndex(int i) {                       // package-private
    if ((i < 0) || (i >= limit))
        throw new IndexOutOfBoundsException();
    return i;
}
// 检查给定的index是否正确
final int checkIndex(int i) {                       // package-private
    if ((i < 0) || (i >= limit))
        throw new IndexOutOfBoundsException();
    return i;
}
// 获取下一个position索引
final int nextGetIndex() {                          // package-private
    int p = position;
    if (p >= limit)
        throw new BufferUnderflowException();
    position = p + 1;
    return p;
}
{% endcodeblock %}

### ByteBuffer中内置方法
    Java17中的实现

- putArray(int index, byte[] src, int offset, int length)

{% codeblock lang:java %}
private ByteBuffer putArray(int index, byte[] src, int offset, int length) {
    // 判断长度是否大于jni复制阀值
    if (((long)length << 0) > Bits.JNI_COPY_FROM_ARRAY_THRESHOLD) {
        long bufAddr = address + ((long)index << 0);
        long srcOffset = ARRAY_BASE_OFFSET + ((long)offset << 0);
        long len = (long)length << 0;
        try {
            // 使用直接内存，将给定的字节数组拷贝进目标的堆外内存块中。最后调用的是copyMemory()方法。其间会检查传入scope的状态
            SCOPED_MEMORY_ACCESS.copyMemory(null, scope(), src, srcOffset, base(), bufAddr, len);
        } finally {
            Reference.reachabilityFence(this);
        }
    } else {
        int end = offset + length;
        for (int i = offset, j = index; i < end; i++, j++) {
            this.put(j, src[i]);
        }
    }
    return this;
}

{% endcodeblock %}

- getArray(int index, byte[] dst, int offset, int length)

{% codeblock lang:java %}
// 大部分实现和put一致，主要是在做内存拷贝时候不一样
private ByteBuffer getArray(int index, byte[] dst, int offset, int length) {
    if (((long)length << 0) > Bits.JNI_COPY_TO_ARRAY_THRESHOLD) {
        long bufAddr = address + ((long)index << 0);
        long dstOffset = ARRAY_BASE_OFFSET + ((long)offset << 0);
        long len = (long)length << 0;
        try {
            // 将当前hb中数据拷贝进传入的字节数组dst中
            SCOPED_MEMORY_ACCESS.copyMemory(scope(), null, base(), bufAddr, dst, dstOffset, len);
        } finally {
            Reference.reachabilityFence(this);
        }
    } else {
        int end = offset + length;
        for (int i = offset, j = index; i < end; i++, j++) {
            dst[i] = get(j);
        }
    }
    return this;
}
{% endcodeblock %}

### 写方法
分类如下:
![bytebuffer_put.webp](/img/bytebuffer_put.webp)

### put(byte)
最简单的写入方法，Java17中，

#### DirectByteBuffer: 直接内存

- put(byte x)

{% codeblock lang:java %}
public ByteBuffer put(byte x) {
    try {
        // scope()会获取到内存片段代理。如果通过上面的方法创建ByteBuffer，这个就为null
        SCOPED_MEMORY_ACCESS.putByte(scope(), null, ix(nextPutIndex()), ((x)));
    } finally {
        Reference.reachabilityFence(this);
    }
    return this;
}
// 大部分方法都会使用，获取
private long ix(int i) {
    return address + ((long)i << 0);
}
{% endcodeblock %}

- put(int i, byte x)

{% codeblock lang:java %}
// 和没有index的相比，offset发生了变化，其它都一样
public ByteBuffer put(int i, byte x) {
    try {
        SCOPED_MEMORY_ACCESS.putByte(scope(), null, ix(checkIndex(i)), ((x)));
    } finally {
        Reference.reachabilityFence(this);
    }
    return this;
}
{% endcodeblock %}

- put(byte[] src, int offset, int length): java17中没有进行重写，默认使用的直接内存

{% codeblock lang:java %}
public ByteBuffer put(byte[] src, int offset, int length) {
    if (isReadOnly())
        throw new ReadOnlyBufferException();
    Objects.checkFromIndexSize(offset, length, src.length);
    int pos = position();
    if (length > limit() - pos)
        throw new BufferOverflowException();
    // 操作直接内存，最后操控就是copyMemory()方法进行拷贝
    putArray(pos, src, offset, length);

    position(pos + length);
    return this;
}

{% endcodeblock %}


- put(ByteBuffer src): 放入ByteBuffer

{% codeblock lang:java %}
public ByteBuffer put(ByteBuffer src) {
    if (src == this)
        throw createSameBufferException();
    if (isReadOnly())
        throw new ReadOnlyBufferException();

    int srcPos = src.position();
    int srcLim = src.limit();
    int srcRem = (srcPos <= srcLim ? srcLim - srcPos : 0);
    int pos = position();
    int lim = limit();
    int rem = (pos <= lim ? lim - pos : 0);

    if (srcRem > rem)
        throw new BufferOverflowException();
    // 操作直接内存
    putBuffer(pos, src, srcPos, srcRem);

    position(pos + srcRem);
    src.position(srcPos + srcRem);

    return this;
}

void putBuffer(int pos, ByteBuffer src, int srcPos, int n) {

    Object srcBase = src.base();
    assert srcBase != null || src.isDirect();
    Object base = base();
    assert base != null || isDirect();

    long srcAddr = src.address + ((long)srcPos << 0);
    long addr = address + ((long)pos << 0);
    long len = (long)n << 0;

    try {
        // 复制
        SCOPED_MEMORY_ACCESS.copyMemory(src.scope(), scope(), srcBase, srcAddr,base, addr, len);
    } finally {
        Reference.reachabilityFence(src);
        Reference.reachabilityFence(this);
    }
}
{% endcodeblock %}

- ByteBuffer putFloat(float value)

{% codeblock lang:java %}
private ByteBuffer putFloat(long a, float x) {
    try {
        // 根据IEEE 754的浮点数“单精度格式”的位布局，返回所指定浮点数的表达形式，保留非数字（NaN）值
        int y = Float.floatToRawIntBits(x);
        // 放入直接内存
        SCOPED_MEMORY_ACCESS.putIntUnaligned(scope(), null, a, y, bigEndian);
    } finally {
        Reference.reachabilityFence(this);
    }
    return this;
}
{% endcodeblock %}

##### SCOPED_MEMORY_ACCESS
从`DirectByteBuffer`的`put()`方法中，我们可以看到直接内存的存储是调用的`SCOPED_MEMORY_ACCESS`的静态方法。

查看源码，我们可以看到它是类`ScopedMemoryAccess`的中的一个静态属性，而且已经初始化，是当前自身类的对象。从注释看，该类定义低级方法来访问堆上和堆外内存。

它对Unsafe类方法的包装，所有方法接受ScopedMemoryAccess.Scope参数，并通过它校验是否可以在安全的方式下访问内存。

单线程下访问内存，不会出问题，当多线程下，提供了对并发访问释放同一块内存区域的管理功能

{% codeblock lang:java %}
public class ScopedMemoryAccess {
    private static final ScopedMemoryAccess theScopedMemoryAccess = new ScopedMemoryAccess();

    public static ScopedMemoryAccess getScopedMemoryAccess() {
        return theScopedMemoryAccess;
    }

    // 放入byte。scope作为参数传入。
    public void putByte(Scope scope, Object base, long offset, byte value) {
        try {
            putByteInternal(scope, base, offset, value);
        } catch (Scope.ScopedAccessError ex) {
            throw new IllegalStateException("This segment is already closed");
        }
    }
    // srcBase 原数据对象，可以是对象、数组（对象），也可以是 Null（为 Null 时必须指定 offset，offset 就是 address）
    // srcOffset 原数据对象的 base offset，如果 srcBase 为空则此项为 address
    // destBase 目标数据对象，规则同 src
    // destOffset 目标数据对象的 base offset，规则同 src
    // bytes 要拷贝的数据大小（字节单位）
    public void copyMemory(Object srcBase, long srcOffset,
                           Object destBase, long destOffset,
                           long bytes) {
        copyMemoryChecks(srcBase, srcOffset, destBase, destOffset, bytes);
        if (bytes == 0) {
            return;
        }
        copyMemory0(srcBase, srcOffset, destBase, destOffset, bytes);
    }

}
{% endcodeblock %}

#### HeapByteBuffer: 堆上分配 put

- put(byte b): 直接放入
{% codeblock lang:java %}
public ByteBuffer put(byte x) {
    // 获取下一个position的值，然后在加offset，最后得到存放位置
    hb[ix(nextPutIndex())] = x;
    return this;
}
{% endcodeblock %}

- put(int i, byte x): 指定index放入
{% codeblock lang:java %}
public ByteBuffer put(int i, byte x) {
    hb[ix(checkIndex(i))] = x;
    return this;
}
{% endcodeblock %}

- put(byte[] src, int offset, int length): java17中，只在HeapByteBuffer中里进行了重写

{% codeblock lang:java %}
public ByteBuffer put(byte[] src, int offset, int length) {
    checkScope();
    Objects.checkFromIndexSize(offset, length, src.length);
    int pos = position();
    if (length > limit() - pos)
        throw new BufferOverflowException();
    // 系统复制，效率高
    System.arraycopy(src, offset, hb, ix(pos), length);
    // 计算新的position
    position(pos + length);
    return this;
}
{% endcodeblock %}

- put(ByteBuffer src): 放入ByteBuffer，和直接内存中的实现一样
- ByteBuffer putFloat(float value)：也和直接内存差不多，应该说Java17之后，基本都是用直接内存
  
{% codeblock lang:java %}
public ByteBuffer putFloat(int i, float x) {
    int x = SCOPED_MEMORY_ACCESS.getIntUnaligned(scope(), hb, byteOffset(checkIndex(i, 4)), bigEndian);
    return Float.intBitsToFloat(x);
}
{% endcodeblock %}

### 读方法

#### DirectByteBuffer

- get(): 最简单的get()方法
- get(byte[] dst): 将当前字节转移到给定的目标字节数组中，具体实现是调用下面的方法。如果读取的长度超出buffer剩余的长度就会抛出异常
- get(byte[] dst, int offset, int length): 同上
{% codeblock lang:java %}
public byte get() {
    try {
        // 直接内存中读取
        return ((SCOPED_MEMORY_ACCESS.getByte(scope(), null, ix(nextGetIndex()))));
    } finally {
        Reference.reachabilityFence(this);
    }
}

public ByteBuffer get(byte[] dst) {
    return get(dst, 0, dst.length);
}
// 默认实现，没有单独的实现方式
public ByteBuffer get(byte[] dst, int offset, int length) {
    Objects.checkFromIndexSize(offset, length, dst.length);
    int pos = position();
    if (length > limit() - pos)
    throw new BufferUnderflowException();
    
    getArray(pos, dst, offset, length);

    position(pos + length);
    return this;
}
{% endcodeblock %}

- get(int i)
{% codeblock lang:java %}
public byte get(int i) {
    try {
        return ((SCOPED_MEMORY_ACCESS.getByte(scope(), null, ix(checkIndex(i)))));
    } finally {
        Reference.reachabilityFence(this);
    }
}
{% endcodeblock %}

- getFloat()
- getFloat(int i)

{% codeblock lang:java %}
public float getFloat() {
    try {
        return getFloat(ix(nextGetIndex((1 << 2))));
    } finally {
        Reference.reachabilityFence(this);
    }
}
public float getFloat(int i) {
    try {
        return getFloat(ix(checkIndex(i, (1 << 2))));
    } finally {
        Reference.reachabilityFence(this);
    }
}

private float getFloat(long a) {
    try {
        // 直接内存中获取
        int x = SCOPED_MEMORY_ACCESS.getIntUnaligned(scope(), null, a, bigEndian);
    return Float.intBitsToFloat(x);
    } finally {
        Reference.reachabilityFence(this);
    }
}
{% endcodeblock %}

#### HeapByteBuffer

- get()
- get(int i)
- get(byte[] dst): 该方法默认实现的都一样，具体不同在于下面的方法
- get(byte[] dst, int offset, int length): hb单独实现
{% codeblock lang:java %}

public byte get() {
    return hb[ix(nextGetIndex())];
}
public byte get(int i) {
    return hb[ix(checkIndex(i))];
}

public ByteBuffer get(byte[] dst, int offset, int length) {
    checkScope();
    Objects.checkFromIndexSize(offset, length, dst.length);
    int pos = position();
    if (length > limit() - pos)
        throw new BufferUnderflowException();
    // 系统拷贝，拷贝数据到dst中
    System.arraycopy(hb, ix(pos), dst, offset, length);
    position(pos + length);
    return this;
}
{% endcodeblock %}

- getFloat(): 获取
- getFloat(int i)

{% codeblock lang:java %}
// 字节偏移量
private long byteOffset(long i) {
    return address + i;
}

public float getFloat() {
    // float占4个字节，所以获取第四个之后的索引的内存的数据
    int x = SCOPED_MEMORY_ACCESS.getIntUnaligned(scope(), hb, byteOffset(nextGetIndex(4)), bigEndian);
    return Float.intBitsToFloat(x);
}

public float getFloat(int i) {
    // 先检查给定的索引是否满足条件，再从缓冲区获取一个float的值
    int x = SCOPED_MEMORY_ACCESS.getIntUnaligned(scope(), hb, byteOffset(checkIndex(i, 4)), bigEndian);
    return Float.intBitsToFloat(x);
}

{% endcodeblock %}

### 简单使用

## 总结

![bytebuffer_put.webp](/img/bytebuffer_defect.webp)

参考：[一文搞懂ByteBuffer使用与原理](https://juejin.cn/post/7217425505926447161)