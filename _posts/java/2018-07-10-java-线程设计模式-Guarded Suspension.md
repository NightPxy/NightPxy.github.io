---
layout: post
title:  "线程设计模式-Guarded Suspension"
date:   2018-07-10 13:31:01 +0800
categories: java
tag: [java]
---

* content
{:toc}



## Guarded Suspension  

核心思想是,是提前准备多个处理线程,然后不断对新进入的请求进行处理  
这个处理过程非常可能出现两种情况:   
* 请求没有或太少,处理线程处于饥饿状态.  
* 请求太多,处理线程处于繁忙状态.  

很多时候,一个线程会基于某种原因需要暂停,并等待一个恢复性的条件之后再次运行.比如将一个任务分解为多个线程协作完成,一个线程必须等待另一个线程完成之后才能继续执行等等   

```scala
object GuardedSuspensionDemoApp {
  def main(args: Array[String]): Unit = {
    val list = List(
      new PushThread("push1"),
      new PushThread("push2"),
      new PushThread("push3"),
      new PopThread("pop1"),
      new PopThread("pop2")
    )
    list.foreach(x => x.start())
  }
}


object RequestQueue {
  private val queue = new mutable.Queue[String]()
  private val lock = new ReentrantLock()
  private val condition = lock.newCondition()

  def push(message: String) = {
    try {
      lock.tryLock(10, TimeUnit.SECONDS)
      queue.enqueue(message)
      condition.signal()
    } finally {
      lock.unlock()
    }
  }

  def pop(): String = {
    try {
      lock.tryLock(10, TimeUnit.SECONDS)
      if (queue.isEmpty) {
        condition.await()
        null
      }
      else {
        queue.dequeue()
      }
    } finally {
      lock.unlock()
    }
  }
}

class PushThread(name: String) extends Thread {
  override def run(): Unit = {
    for (i <- 1 to 5) {
      RequestQueue.push(s"$name+$i")
      Thread.sleep(100)
    }
  }
}

class PopThread(name: String) extends Thread {
  override def run(): Unit = {
    while (true) {
      val message = RequestQueue.pop()
      if (message != null) {
        println(message)
      }
      else {
        println("null")
      }
      Thread.sleep(1000)
    }
  }
}
```