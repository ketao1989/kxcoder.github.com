---
layout: post
category: Java
title:  "Java一些小Tips"
date: 2014-11-24 22:21:35 +0800
comments: true
---

插播广告：ubuntu系统的sublime text 3 中文无法输入，需要修复一些so库。参考：<http://jingyan.baidu.com/article/f3ad7d0ff8731609c3345b3b.html>

## 目录

1. [前言](#Intro)
1. [java Integer并发问题](#Integer)
1. [String 可变性方法](#StringChange)
1. [volatile 非线程安全性](#VolatileUnsafe)


## <a id="Intro">前言</a>

虽然java开发已经快两年了，但是对于java内部一些小的技巧和坑还是会有些不了解。这里记录下。

<!--more-->

## <a id="Integer">java Integer并发问题</a>

前段时间看书，顺带提到说`Integer.valueOf( )会导致死锁问题`很是惊讶。于是，查看了JDK源码，果然如此。

JDK代码对把-128 到127 之间的整数转换成`Integer`的时候，并不会new一个新的Integer对象，而是从 内部的`IntegerCache`中直接获取已经创建好的对象（第一次调用时会创建这个`IntegerCache`）。

>> 由于内部`IntegerCache`共用，所以在不同的地方对同一个数值调用`valueOf`获取cache中同一个对象，这样很可能会导致死锁。此外，关于整数范围可以使用VM初始设置（-XX:AutoBoxCacheMax=<size>，但是不能比127小）

### valueOf方法实现代码

`valueOf`方法实际上就是调用`IntegerCache`获取对应下标的对象，而`IntegerCache`实际上就是一个Integer对象数组。关于`IntegerCache`实现如下所示：

{% codeblock lang:java valueOf方法实现代码 %}

private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1); 
            }
            high = h;

            cache = new Integer[(high - low) + 1];//大小为正负两边总的数量
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);
        }

        private IntegerCache() {}
    }

{% endcodeblock %}

>> 之所以会导致死锁，主要原因是因为当两个线程不断的调用valueOf时，比如一个为 `Integer a=Integer.valueOf(10) + Integer.valueOf(20)`，另外一个线程调用`Integer b= Integer.valueOf(20) +Integer.valueOf(10)`，while中不停的调用，就可能出现死锁异常。
>> 
>> 

## <a id="StringChange">String 可变性方法</a>

### String属性

在一般的代码逻辑中，String 显然是不可以改变的，其实际上是一个final属性的值类型。也就是说，我们申明一个变量为String 常量，只要不替换变量的String引用，其具体字符串内容是不会改变的。但是，一个有趣的方法可以改变这些。

###改变String 值

{% codeblock lang:java 改变String %}

/*************************************************************************
    > File Name: StringChange.java
    > Author: ketao1989
    > Mail: ketao1989@gmail.com
    > Created Time: 2014年12月10日 星期三 20时48分37秒
 ************************************************************************/
import java.lang.reflect.Field;

public class StringChange {
    
    private static Field valueField;
    
    static {
        try {
            valueField = String.class.getDeclaredField("value");
            if (valueField != null) {
                valueField.setAccessible(true);                                                            
            }
            // (1)
            valueField.set("Immutable String", "Change Now！".toCharArray()); 
                                                
        } catch (Exception e) {
            e.printStackTrace();
                                    
        }
                
    }
    
    public static void main(String[] args) {
        System.out.println("Immutable String");
    }

}

{% endcodeblock %}

>> Note: 其实就是一个反射特性的有趣应用。其内部运行流程为：当运行(1)的时候，应用会去String池中寻找该字符串常量对应的引用，如果没有则新建一个，并且会把引用对应的值更改为新设置的值`Change Now！`。
>> 
>> 然后在程序运行main的时候，会去String池中寻找`Immutable String`对应的引用，但是这个引用内部的值以及被更改为`Change Now！`，所以打印的时候就会输出`Change Now！`.
>> 

## <a id="VolatileUnsafe">volatile 非线程安全性</a>

### volatile 属性

`volatile` 在java中是一种力度比较轻的synchronized，使用简单，相应编码也比较少；显然，其对应的同步功能也会比synchronized少一部分。

在一般地代码使用过程中，经常会存在一些错误的使用方法，由于涉及多线程的方面的使用，有时候测试不严谨，就会存在一些线上不易发现的错误数据。

`volatile`变量保证的是，定义属性字段在java内存模型中数据一致性，也就是说，当其他线程操作更新了该属性字段，所有读取该属性字段的线程都会去全局内存上去获取最新值。关于内存模型一致性参考下面这张图就可以明白了，文章可以看看改图对应的博文：[深入理解Java内存模型](http://www.infoq.com/cn/articles/java-memory-model-1)

<img src="/images/2014/11/jmm.png" />

### volatile 非线程安全

虽然，`volatile`虽然保证内存一致性，但是并不是线程安全的，这两个特性是不对应的。下面给出一个代码说明：

{% codeblock lang:java volatile 非线程安全 %}

/**
 * @author: ketao Date: 14/12/14 Time: 下午8:21
 * @version: \$Id$
 */
public class VolatileUnsafe {

    static volatile int COUNT = 0;

    public static void main(String[] args) {

        for (int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {

                    stopSecond(1);
                    COUNT += 100;
                }
            }).start();
        }

        stopSecond(1000);
        System.out.println("100 times increase sum is :" + COUNT);
    }

    private static void stopSecond(int a) {

        try {
            Thread.sleep(a * 10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

{% endcodeblock %}

代码写的很简单，但是足以说明 `volatile`确实是无法保证线程安全性的。

`volatile`确实可以保证线程变量内存一致性，但是对于上述这种多线程对变量更新操作的问题是无法保证的。在代码执行的时候，每个线程启动有先后，但是在sleep的时候，很多线程都取得了最新的值，这些值计算完了之后会写入内存，但是这个时候如果变量值改变了，线程并不知道，因为`volatile`只能保证读变量时是内存中最新的值，写的时候写入共享内存中；但是无法保证写入的时候去判断先前获取的值是否还是计算完之后还是最新的，因此也就无法保证线程安全性。

综上所述，`volatile`是无法保证线程安全性的；如果有这种需求，请使用`AtomicXxx`或者`synchronized`属性来实现。

此外，关于`volatile`变量一些优点和使用技巧，请参考IBM计数博客：[Java 理论与实践: 正确使用 Volatile 变量](http://www.ibm.com/developerworks/cn/java/j-jtp06197.html)