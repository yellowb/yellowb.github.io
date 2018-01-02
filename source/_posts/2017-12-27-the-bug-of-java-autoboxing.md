---
title: 关于Java自动装箱/拆箱的一个坑
date: 2017-12-27 16:52:33
tags:
- Java
---
# 背景

相信对于用过一段时间Java的使用者来说，自动装箱（Autoboxing）和自动拆箱（Autounboxing）应该不是什么陌生的概念了。简单来说就是Java为基本数据类型（int、long、double等·）提供了对应的包装类（Integer、Long、Double等），谁让“Java中一切皆为对象呢”？（滑稽）。
依赖自动装箱/拆箱的特性，使得在写代码的时候，可以用比较简洁的方式混合使用基本数据类型和包装类，比如：

```java
// 啰嗦写法
Integer d = Integer.valueOf(1);
Integer e = Integer.valueOf(d.intValue() + 1);

// 简洁写法
Integer d = 1;
Integer e = d + 1;
```

可以看出，自动装箱/拆箱实际上就是官方提供的一个语法糖，编译器会在背后会自动生成基本数据类型和包装类互转换的代码。这里建议大家读读Java中[Integer.java的源码](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/Integer.java "Integer.java的源码")，会发现其它奇妙的地方。

# 坑
既然官方给我们提供了这么方便的特性，那么我们用起来是不是就万事大吉了呢？答案是否定的，不小心的话会踩进一些奇怪的坑中，比如下面这2行代码，大家可以先猜猜输出结果是啥：

```java
System.out.println(((Long) 1L) == 1);
System.out.println(new Long(1).equals(1));
```
可能初次看有点复杂，以下结合反编译后的结果解释里面发生了什么。对于第一行代码，我们来看其字节码：
```java
// 前三行是等号左边
// 把long常量1压入操作数栈 
LCONST_1
// 调用Long.valueOf函数并把刚刚的常量1穿进去作为参数，返回一个包装对象Long(1)
INVOKESTATIC java/lang/Long.valueOf (J)Ljava/lang/Long;
// 在这个包装对象Long(1)上调用longValue函数，获得其内部的long值1
INVOKEVIRTUAL java/lang/Long.longValue ()J

// 后一行是等号右边
// 把long常量1压入操作数栈（可见等号右边的int 1已经被自动提升成long 1）
LCONST_1
// 比较这两个值
LCMP
```
可以看到，等号左边是一个自动装箱再拆箱的过程，得到的值就是long 1。等号右边的int 1被自动提升成long 1，所以等号左右是相等的。

对于第2行代码，这个坑更大了，还是先看字节码：
```java
// 头两行，是调用Long的构造函数，对于到new Long(1)
LCONST_1
INVOKESPECIAL java/lang/Long.<init> (J)V

// 注意！！！接下来两行调用Integer的包装类生成了一个Integer(1)，而不是Long！
ICONST_1
INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;

// 调用Long(1)的equals函数，把Integer(1)传进去
INVOKEVIRTUAL java/lang/Long.equals (Ljava/lang/Object;)Z
```
可见对于equals函数的输入参数，“1”被自动装箱成Integer对象了。接下来看看Long.equals函数里是怎么实现的:

```java
    /**
     * Compares this object to the specified object.  The result is
     * {@code true} if and only if the argument is not
     * {@code null} and is a {@code Long} object that
     * contains the same {@code long} value as this object.
     *
     * @param   obj   the object to compare with.
     * @return  {@code true} if the objects are the same;
     *          {@code false} otherwise.
     */
    public boolean equals(Object obj) {
        if (obj instanceof Long) {  // 由于传入的不是Long对象，所以返回false！
            return value == ((Long)obj).longValue();
        }
        return false;
    }
```
可以看到坑爹的地方在于if判断，由于我们传入的是Integer对象，所以直接返回了false！如果滥用自动装箱/拆箱，并且不清楚源码的实现，很可能会出现一些很难查的bug！

为了使其返回true，可以修改代码如下：
```java
System.out.println(new Long(1).equals(1L));
```
强制传入1L，自动装箱就能变成Long类型了。

# 总结
对于long值，还是写成xxL的形式比较好，这样能保证自动装箱/拆箱时的准确！





