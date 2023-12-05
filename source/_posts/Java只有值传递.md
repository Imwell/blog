    ---
title: Java只有值传递
date: 2022-10-09 19:21:53
tags:
    - Java
categories:    
    - Java
description: 为什么 Java 中只有值传递？
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## 为什么Java只有值传递

搞清楚之前，让我们先了解下
1. 什么是形参&实参

{% codeblock lang:java %}
public static void main(String[] args) {
    Person p1 = new Person("123");
    Person p2 = new Person("456");
    swap(p1, p2); // p1, p2 是实参
}

public void setName(String name) { // name为形参
    this.name = name;
}

{% endcodeblock %}

2. 什么是值传递&引用传递

**值传递**: 方法接收的是实参值的拷贝，会创建副本
**引用传递**: 方法接收的直接是实参所引用的对象在堆中的地址，不会创建副本，对形参的修改将影响到实参。

## 为什么Java中只有值传递？
废话不多说，我们直接从案例看起

案例1：
{% codeblock lang:java %}
public static void main(String[] args) {
    int i = 10;
    count(i);
    System.out.println("i:" + i); // i: 10
}

private static void count(int j) {
    j = 20;
    System.out.println("j:" + j); // j: 20
}
{% endcodeblock %}
从结果看，i的值并没有因为j的修改而变换。

案例2：
{% codeblock lang:java %}
public static void main(String[] args) {
    Person p1 = new Person("123");
    Person p2 = new Person("456");
    p1.change(p2);
    System.out.println("p:" + p1.name); // p: 456
}

public void change(Person person) {
    this.name = person.name;
    System.out.println("p change:" + this.name);
}
{% endcodeblock %}
从结果看，p1的name发生了变化，那么是不是我们的结论就有误呢？答案当然不是的。
p1的name变化并不是因为调用`change()`方法传递参数的方法从值传递变为引用传递，而是因为方法内对象的name变量，由于进行重新赋值，有了新的引用而发生了变化，这个和Java参数方法传递的方式并无关系。

> 所以说，Java中其实还是值传递的，只不过对于对象参数，值的内容是对象的引用。

案例3：
{% codeblock lang:java %}
public static void main(String[] args) {
    Person p1 = new Person("123");
    Person p2 = new Person("456");
    swap(p1, p2);
    System.out.println("p1:" + p1); // p1:Person{name='123'}
    System.out.println("p2:" + p2); // p2:Person{name='456'}
}

private static void swap(Person p1, Person p2) {
    Person p = p1;
    p1 = p2;
    p2 = p;
    System.out.println("p1:" + p1); // p1:Person{name='456'}
    System.out.println("p2:" + p2); // p2:Person{name='123'}
}
{% endcodeblock %}
从结果看，p1和p2的交换也没有影响到原实参，内部交换的只是拷贝实参的地址

## 总结：
Java 中将实参传递给方法（或函数）的方式是 值传递 ：
1. 如果参数是基本类型，传递的值就是字面量的拷贝，会创建副本
2. 如果蚕食引用类型，传递的是对象在堆中地址值的拷贝，会创建副本
