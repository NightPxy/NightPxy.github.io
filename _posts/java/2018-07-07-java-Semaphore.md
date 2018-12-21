---
layout: post
title:  "java-Semaphore"
date:   2018-07-07 13:31:01 +0800
categories: java
tag: [java,多线程]
---

* content
{:toc}

## 概述    

信号量(Semaphore),是一个针对一组线程的信号量许可集(本质是一个数字).  
当有一个线程想要执行时,需要首先向信号量申请一个许可(减一但不得归0),如果能拿到许可线程就可以正常执行,如果不能拿到许可就必须阻塞等待.而线程完成后可以上缴许可(加一),之后信号量会通知其它阻塞线程解除阻塞继续来申请许可  

信号量不关心哪个线程来申请许可,也不关心每个线程的执行进度.它只是一个栈门,限制同一时间运行的线程数量  

信号量通常用于限制访问某些资源的线程数量,比如数据库连接,用信号量设置一个最大同时访问数据库数量来防止无限撑爆数据库连接  

Semaphore 注意点  
* 必须保证在任何情况下信号量许可使用完毕后上缴,否则最终信号量会因为耗尽许可而阻塞全部线程  
* 必须保证申请了多少许可就上缴多少许可  
如果少上缴了,信号量会因为回收不到足够的耗尽许可而阻塞全部线程  
如果多上缴了,会让信号量控制失效(因为多发了许可)  

## Demo  

```scala
object SemaphoreDemoApp {
  def main(args: Array[String]): Unit = {


    val lock = new Semaphore(2)

    val threads = List(
      new Thread(new ProcessWait("t1", lock)),
      new Thread(new ProcessWait("t2", lock)),
      new Thread(new ProcessWait("t3", lock)),
      new Thread(new ProcessWait("t4", lock)),
      new Thread(new ProcessWait("t5", lock))
    )

    threads.foreach(x => x.start())
    println("开始执行")

  }
}

class ProcessWait(info: String, lock: Semaphore) extends Runnable {
  override def run(): Unit = {
    try {
      lock.acquire()
      Thread.sleep(1000); //模拟执行等待
      println(s"$info 模拟执行完毕")
    } finally {
      lock.release(1)
    }
  }
}
```
