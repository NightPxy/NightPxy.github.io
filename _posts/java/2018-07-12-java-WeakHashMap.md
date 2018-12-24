---
layout: post
title:  "java-WeakHashMap"
date:   2018-07-12 13:31:01 +0800
categories: java
tag: [java]
---

* content
{:toc}

## 概述  

WeakHashMap 跟HashMap非常类似,也是一个散列表,并且允许K ,V 为Null  
WeakHashMap 的最大特点,键是弱引用  

弱引用,是GC回收的一个概念.也就是弱引用可以正常的指向某个对象,但不会视为GC可达性判定,也就是随着GC之后,只被弱引用引用的键会被回收  


WeakHashMap弱引用键是个非常特殊的地方  
* 调用两次Size,可能结果不同(有锁也一样,下同)  
* isEmpty,可能上一句为false下一句就为true了  
* containsKey 可能上一句判定为真,下一步get就是null  

## 回收原理  

一个正常的联想,如果静态WeakHashMap的键被GC回收,那么键对应的值会不会造成内存泄露?     
答案是会的,因为键回收后,本身还维持着WeakHashMap对值的强引用,此时的Value是不会被GC,也就是通常意义的内存泄露了  

但WeakHashMap会不会产生泄露呢.答案是不会,原因就在与WeakHashMap的回收机制  
因为键已经被GC了,所以已经无法从键去定位到值,为此WeakHashMap的解决思路是ReferenceQueue  
ReferenceQueue是一个JDK内置的集合,它的意义只在如果一个对象引用了ReferenceQueue,那么JVM保证GC时会将该对象回推其引用的ReferenceQueue  

利用ReferenceQueue这种机制,WeakHashMap的回收机制如下  
* 每一个键值对加入时都再同时加入一个ReferenceQueue  
* 当弱键被回收时,ReferenceQueue保证弱键首先会回到ReferenceQueue(此时弱键依然存在,ReferenceQueue本身是一种强引用存在)  
* 在每次使用WeakHashMap时先同步ReferenceQueue  
即先从ReferenceQueue取出所有应该回收的键值对清理,这样就能保证值始终被清理了,之后再从WeakHashMap去删除键,这样键值都会失去强引用,在下次GC时就会被正常清理了  
 