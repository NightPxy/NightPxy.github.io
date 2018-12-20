---
layout: post
title:  "java-ThreadLocal"
date:   2018-07-04 13:31:01 +0800
categories: java
tag: [java,多线程]
---

* content
{:toc}

## 概述    

ThreadLocal是一种用资源隔离的形式(为每个线程分配副本)解决的线程安全  


## 源码  

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

ThreadLocal最核心的主题就是`WeakReference<ThreadLocal<?>>`.也就是说ThreadLocal的本质是一个以线程对象为键,值中再包含一个Hash的弱引用大Hash  


## ThreadLocal内存泄露  

有很多描述都说ThreadLocal会引起内存泄露,理由如下  
从源码可以看出ThreadLocal的本质是以线程对象的弱引用为键的大Hash.理由就是这个弱引用.弱引用如果被GC后,就无法从以这个弱引用为键从ThreadLocal读取了.但是实际其中的值又继续存在其中,并且因为这个引用关系也无法回收.这就构成内存泄露了  

这个说法对,也不对.现在来详细分析一下这里  

首先,焦点问题在这个弱引用.那么换成强引用行不行?  
很可惜答案是不行.因为ThreadLocal的本体是一个`static class Entry`,如果是强引用,那么当线程对象本身会因为ThreadLocal始终维持了一个引用而无法回收.这本身就会造成内存泄露   
使用弱引用的目的就在这里,JDK期望当线程消亡时伴随Thread线程对象的消失时,ThreadLocal中也能自动被GC  

其次,线程对象因为弱引用被GC后,ThreadLocal中的值会不会有问题  
很遗憾,从个人的理解上这里的确是有点问题,但不至于是内存泄露这么严重  
事实上,JDK自己已经意识到这里有问题了.它们对此采取了一个补救措施,就是在每次get的时候都会从尽可能的删除null键.(只要删除null键就会失去对其引用就会被正常GC了),但这个补救措施其实相当弱  

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
     Entry[] tab = table;
     int len = tab.length;

     while (e != null) {
         ThreadLocal<?> k = e.get();
         if (k == key)
             return e;
         if (k == null)
             expungeStaleEntry(i);
         else
             i = nextIndex(i, len);
         e = tab[i];
     }
     return null;
 }
``` 

* 首先从ThreadLocal的直接索引位置(通过ThreadLocal.threadLocalHashCode & (len-1)运算得到)获取Entry e，如果e不为null并且key相同则返回e
* 如果e为null或者key不一致则向下一个位置查询，如果下一个位置的key和当前需要查询的key相等，则返回对应的Entry，否则，如果key值为null，则擦除该位置的Entry，否则继续向下一个位置查询  
这补救措施很弱是因为只有读取到null才会向下遍历清除null键,并且这种遍历一旦匹配就会停止  
这意味着这个补救措施只能挽回一部分的泄露内存.  

最好的解决办法是开发者自己来  
* ThreadLocal是提供remove方法的,最好的解决办法是.如果自己不再需要时手动remove掉  
* 