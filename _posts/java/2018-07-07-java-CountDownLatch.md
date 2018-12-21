---
layout: post
title:  "java-CountDownLatch"
date:   2018-07-07 13:31:01 +0800
categories: java
tag: [java,多线程]
---

* content
{:toc}

## 概述    


CountDownLatch 是在完成一个或一组线程之前,让一个或一组线程阻塞等待  

CountDownLatch的机制是

CountDownLatch 是一个常常用来与CyclicBarrier比较的同步辅助器  
* CountDownLatch是一次性的,归0后不可重置,而CyclicBarrier可以在归0后Reset继续下一轮  
* CountDownLatch只会阻塞之后线程,之前的线程无论是否CountDownLatch完成都会继续执行不会阻塞.CyclicBarrier会阻塞之前线程,并在最后解除是全部解除阻塞  

综上,CyclicBarrier一般用在WorkMaster模式中,CountDownLatch则一般用于并行加载一些耗时长的资源.比较常见的场景如使用ZK和Kafka的场景,客户端连接一次都比较费时.可以使用两个子线程去并行连接并同时阻塞主线程,在ZK和Kafka都连接成功后解除主线程阻塞就可以进入后续流程了  

## Demo  

```scala

object ProcessWaitDemoApp {
  def main(args: Array[String]): Unit = {

    val lock = new CountDownLatch(3)

    val threads = List(
      new Thread(new ProcessWait("t1",lock)),
      new Thread(new ProcessWait("t2",lock)),
      new Thread(new ProcessWait("t3",lock))
    )

    threads.foreach(x=>x.start())
    println("开始执行")
    lock.await()

    println("前置线程全部执行完毕")
  }
}

class ProcessWait(info:String,lock:CountDownLatch) extends Runnable {

  override def run(): Unit = {
     Thread.sleep(5000); //模拟执行等待
     println(s"$info 模拟执行完毕")

    lock.countDown()
  }
}
```

## 实现原理  


