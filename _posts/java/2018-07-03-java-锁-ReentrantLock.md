---
layout: post
title:  "java-锁-ReentrantLock"
date:   2018-07-01 13:31:01 +0800
categories: java
tag: [java,多线程]
---

* content
{:toc}

## 概述    

java.util.concurrent.lock.ReentrantLock 是一个可重入的互斥锁.  

传统的synchronized的同步并不完美.最大问题在于无法中断正在获取锁的线程(也就是一旦开始等待,想不等都做不到了)  

所以JDK才引入了 java.util.concurrent.lock.ReentrantLock,它拥有与synchronized相同的语义,但添加了定时锁等候和可中断等非常有用的东西

## 最重要的地方  

ReentrantLock的必须显式释放,也就是最好包裹在finally中,如果忘记释放ReentrantLock后果将是毁灭性的,这一点与synchronized不同,切记切记 

```java
Lock lock = new ReentrantLock();  
lock.lock();  
try {   
  // update object state  
}  
finally {  
  lock.unlock(); 
  //切记不能忘记释放锁
}  
```

## 公平与非公平  

构造 ReentrantLock(boolean fair),指定可公平可不公平  

## Condtion  

在ReentrantLock中可以配合使用Condition来完成线程阻塞与唤醒功能  

```java
private Lock lock = new ReentrantLock() // 独占锁
private Condition fullCondtion = lock.newCondition() // 生产条件

...
if(XXX)
fullCondtion.await() //通过条件await阻塞线程
...//其它线程
fullCondtion.signal();//通过条件signal唤醒线程  

```

## 可中断等待   

```java
// 如果锁在给定等待时间内没有被另一个线程保持，且当前线程未被中断，则获取该锁。
boolean tryLock(long timeout, TimeUnit unit)
```





