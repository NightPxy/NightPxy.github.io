---
layout: post
title:  "java-CyclicBarrier"
date:   2018-07-07 13:31:01 +0800
categories: java
tag: [java,多线程]
---

* content
{:toc}

## 概述    


CyclicBarrier 是让一组线程互相等待直到最后的屏障点,其机制设定多个屏障点.在最后的屏障点之前全部线程并行执行,之后全部线程阻塞直到收到解除阻塞通知  
CyclicBarrier 在最后屏障点完成之后,可以重置重新使用.所以也称为可循环屏障  
CyclicBarrier 有个劣势是无法动态变更屏障点数量.所以在某些动态MasterWorker模式下效果不好 

CyclicBarrier是一个整体概念,任意一个线程发生问题都会整体退出CyclicBarrier并将其设为损坏  
CyclicBarrier使用时尽量手动处理下两个异常 InterruptException 和 BrokenBarrierException   
* 当前线程在进入之前被中断,将会抛出InterruptException并清除当前线程中断状态.同时其它线程将全部抛出BrokenBarrierException并将CyclicBarrier设为损坏状态.  
* 如果当前线程在运行时抛出异常,将会将异常传递到当前线程并将CylicBarrier设为损坏  


## MasterWorkerDemo  

```scala

object MasterWorkerDemoApp {
  def main(args: Array[String]): Unit = {
    val data = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val master = new Thread(new Master(data))
    master.start()
  }
}

class Master(datas: List[Int]) extends Runnable {

  private val cb = new CyclicBarrier(3)

  case class WorkerState(var result: Int)

  class Worker(data: List[Int], state: WorkerState) extends Runnable {
    override def run(): Unit = {

      data.foreach(item => {
        state.result += item
        Thread.sleep(100);
      })

      try {
        cb.await()
      }
      catch {
        case e: InterruptedException => state.result = 0
        case e: BrokenBarrierException => state.result = 0
      }
    }
  }

  override def run(): Unit = {
    val dataSplitTupple = datas.partition(x => x % 3 == 0)
    val dataSplits = List(dataSplitTupple._1, dataSplitTupple._2);

    val dataSplitWorker = dataSplits.map(dataSplit => {
      val workerState = WorkerState(0)
      (new Worker(dataSplit, workerState), workerState)
    })

    dataSplitWorker.foreach(x => new Thread(x._1).start())

    try {
      cb.await()
    }
    catch {
      case e: InterruptedException =>  println("c"+e.getMessage)
      case e: BrokenBarrierException => println("d"+e.getMessage)
    }

    if (!cb.isBroken) {
      val sum = dataSplitWorker.map(x => x._2.result).sum
      println(sum)
    }
    else {
      println("error")
    }
    cb.reset()
  }
}

```

## 实现原理  

CyclicBarrier 核心在于计数器(parties总数,count当前数)+锁( ReentrantLock 和 Condition).全部线程执行后基于ReentrantLock.Condition阻塞await,直到当前计数达到总数后触发接触阻塞  

```java
...

// 阻塞实现是基于 ReentrantLock 和 Condition
private final ReentrantLock lock = new ReentrantLock();
private final Condition trip = lock.newCondition();

// 阻塞条件就是这个两个计数
private final int parties;  //屏障总数
private int count; //当前越过的屏障数  

// CyclicBarrier 状态
// 可恢复实现 = 重置状态 + Condition广播全体解除阻塞  
private Generation generation = new Generation();

// CyclicBarrier的核心处理方法  
private int dowait(boolean timed, long nanos)
	throws InterruptedException, BrokenBarrierException,
		   TimeoutException {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		final Generation g = generation;

        //判断锁状态(CyclicBarrier共享),损坏抛出BrokenBarrierException
		if (g.broken)
			throw new BrokenBarrierException();
        // 判断当前线程状态
        // 如果中断(Thread.interrupted中断,获取后重置)
        // 如果当前线程中断首先是退出锁和更新锁状态然后以InterruptedException抛出
		if (Thread.interrupted()) {
			breakBarrier();
			throw new InterruptedException();
		}

		int index = --count;
		// 已越过全部屏障,退出锁和更新锁状态(finally)
		if (index == 0) {  
			boolean ranAction = false;
			try {
				final Runnable command = barrierCommand;
				if (command != null)
					command.run();
				ranAction = true;
				nextGeneration();
				return 0;
			} finally {
				if (!ranAction)
					breakBarrier();
			}
		}

		// 如果没有越过全部屏障,当前线程阻塞直到,线程中断,收到解除通知或超时
		for (;;) {
			try {
				if (!timed)
					trip.await();
				else if (nanos > 0L)
					nanos = trip.awaitNanos(nanos);
			} catch (InterruptedException ie) {
				if (g == generation && ! g.broken) {
					breakBarrier();
					throw ie;
				} else {
					// We're about to finish waiting even if we had not
					// been interrupted, so this interrupt is deemed to
					// "belong" to subsequent execution.
					Thread.currentThread().interrupt();
				}
			}

			if (g.broken)
				throw new BrokenBarrierException();

			if (g != generation)
				return index;

			if (timed && nanos <= 0L) {
				breakBarrier();
				throw new TimeoutException();
			}
		}
	} finally {
		lock.unlock();
	}
}
```
