+++
title= "java单例模式"
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

#### 一、饿汉式:

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

#### 二、懒汉式-双重锁检查:

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
#### 三、懒汉式-静态内部类:

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
