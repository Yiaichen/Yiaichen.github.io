---
title: 一个栗子搞懂单例模式
date: 2018-10-06 21:02:41
tags: [设计模式]
categories: 设计模式
---

## 概念
> 单例模式，是一种常用的软件设计模式。在它的核心结构中只包含一个被称为单例的特殊类。通过单例模式可以保证系统中，应用该模式的类一个类只有一个实例。即一个类只有一个对象实例。

### 三种单例模式
- 饿汉式
> 系统加载时初始化实例，即使不加载也会初始化，占用内存较大，线程安全

- 懒汉式
> 系统加载不初始化，需要加载实例时再初始化实例，线程不安全，加了双重检查之后线程安全

- 枚举
> JDK1.5加入的，也算是最推荐使用的方法，兼顾内存跟线程安全


## 举个栗子
### 懒汉式
``` java
/**
 * 单例模式 (饿汉式)
 */
public final class HungerSingleton {

    /**
     * 杜绝外面直接new 只有一种获取方式
     */
    private HungerSingleton() {}

    /**
     * 实例化
     */
    private static final HungerSingleton INSTANCE = new HungerSingleton();

    /**
     * 获取实例
     */
    public static HungerSingleton getInstance() {
        return INSTANCE;
    }

}
```

- 优点：单例占用内存比较小，初始化时就会被用到的情况。
- 缺点：单例占用的内存比较大，或单例只是在某个特定场景下才会用到

### 懒汉式
``` java
/**
 * 单例模式 (懒汉式)
 */
public final class LazySingleton {

    /**
     * 杜绝外面直接new 只有一种获取方式
     */
    private LazySingleton() {}

    /**
     * 实例化
     */
    private static volatile LazySingleton INSTANCE;

    /**
     * 获取实例
     */
    public static LazySingleton getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new LazySingleton();
        }
        return INSTANCE;
    }

}
```

- 优点：内存节省，由于此种模式的实例实在需要时创建，如果某次的程序运行没有用到，就是可以节省内存
- 缺点：线程不安全，分析见下面问题

``` java
/**
 * 单例模式 (懒汉式)
 */
public final class LazySingleton {

    /**
     * 杜绝外面直接new 只有一种获取方式
     */
    private LazySingleton() {}

    /**
     * 实例化
     */
    private static volatile LazySingleton INSTANCE;

    /**
     * 获取实例（双重检查）
     */
    public static LazySingleton getInstance() {
        if (INSTANCE == null){
            synchronized (LazySingleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new LazySingleton();
                }
            }
        }
        return INSTANCE;
    }

}
```
- 优点：多线程安全

- 缺点：执行效率低，每个线程在想获得类的实例时候，执行getInstance()方法都要进行同步。而其实这个方法只执行一次实例化代码就够了，后面的想获得该类实例，直接return就行了。方法进行同步效率太低要改进。

``` java
/**
 * 枚举是天然单例
 */
public enum EnumSingleton {
    INSTANCE;


    @Override
    public String toString() {
        return super.toString();
    }
}
```
- 优点：兼顾内存和多线程安全
- 缺点：为啥没有早点遇到你（1.5版本之后更新）


## 问题
> 为什么要考虑线程安全？

### 举个栗子（懒汉式非双重检查）

| 步骤 | 线程1 | 线程2 |
| :---------:| :---------: | :-------: |
| 1 |  getInstance() |   |
| 2 |   |  getInstance() |
| 3 | if (INSTANCE == null) |   |
| 4 |   | if (INSTANCE == null) |
| 5 | INSTANCE = new Singleton(); | |
| 6 | return INSTANCE; |   |
| 7 | | INSTANCE = new Singleton(); |
| 8 |   | return INSTANCE; |

> 这里就发生的线程安全的问题，1、2两个步骤分别由线程1、2进入getInstance()的方法，然后3、4两个步骤同时通过，因为这个时候确实还没有实例化为null，所以后面就会线程1`new`一个，线程2也new一个INSTANCE，这样就违背了单例的原则，所以考虑线程安全还是有必要的。