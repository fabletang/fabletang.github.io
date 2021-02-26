+++
title= "java模式:单例/多例/线程单例"
description= "java singleten pattern"
date = "2021-02-26"
tags = [
    "java"
]
categories = [
  "技术"
]
series = [
  "编程"
]
+++
 java单例模式有5种实现方式,推荐以下3种方式. 

#### 一、单例-饿汉式:

特点:线程安全，不能延时加载
```java
public enum SingletonDemo1 {
      
     //枚举元素本身就是单例
     INSTANCE;
      
     //添加自己需要的操作
     public void singletonOperation(){     
     }
 }
```

#### 二、单例-懒汉式-双重锁检查:

特点:线程安全，延时加载。
ps: java1.5后,volatile可以避免jvm初始化内存分配可能乱序的问题。

```java
public class SingletonDemo2 {
         private static volatile SingletonDemo2 SingletonDemo2;
  
         private SingletonDemo2() {
         }
  
         public static SingletonDemo2 newInstance() {
             if (SingletonDemo2 == null) {
                 synchronized (SingletonDemo2.class) {
                     if (SingletonDemo2 == null) {
                         SingletonDemo2 = new SingletonDemo2();
                     }
                 }
             }
             return SingletonDemo2;
         }
     }

```
#### 三、单例-懒汉式-静态内部类:

特点:线程安全，延时加载。

```java
public class SingletonDemo3 {
      
     private static class SingletonClassInstance{
         private static final SingletonDemo3 instance=new SingletonDemo3();
     }
      
     private SingletonDemo3(){}
      
     public static SingletonDemo3 getInstance(){
         return SingletonClassInstance.instance;
     }
      
 }

```

#### 四、多例:
多例模式，我们可以将类的实例都编上号，然后将实例存放在一个Map:

```java
public class MultiInstance {
    // 实例编号
    private long instanceNum;

    // 用于存放实例
    private static final Map<Long, MultiInstance> ins = new HashMap<>();

    static {
        // 存放 3 个实例
        ins.put(1L, new MultiInstance(1));
        ins.put(2L, new MultiInstance(2));
        ins.put(3L, new MultiInstance(3));
    }

    private MultiInstance(long n) {
        this.instanceNum = n;
    }

    public MultiInstance getInstance(long n) {
        return ins.get(n);
    }
}
```
实际上,Java 中的枚举就是一个“天然”的多例模式,其中的每一项代表一个实例:

```java
public enum MultiInstance {
    ONE,
    TWO,
    THREE;
}
```
#### 五、线程中的单例:
一般情况下，我们所说的单例的作用范围是进程唯一的，就是在一个进程范围内，一个类只允许创建一个对象，进程内的多个线程之间也是共享同一个实例。
那么线程唯一的单例就是，一个实例只能被一个线程拥有，一个进程内的多个线程拥有不同的类实例。
我们同样可以用 ConcurrentHashMap 来实现:

```java
public class ThreadSingleton {
    private static final ConcurrentHashMap<Long, ThreadSingleton> instances
            = new ConcurrentHashMap<>();

    private ThreadSingleton() {}

    public static ThreadSingleton getInstance() {
        Long id = Thread.currentThread().getId();
        instances.putIfAbsent(id, new ThreadSingleton());
        return instances.get(id);
    }
}
```
