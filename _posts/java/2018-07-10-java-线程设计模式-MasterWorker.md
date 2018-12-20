---
layout: post
title:  "线程设计模式-MasterWorker"
date:   2018-07-10 13:31:01 +0800
categories: java
tag: [java]
---

* content
{:toc}



## MasterWorker  

MasterWorker 也是常用的线程设计模式  
MasterWorker 模式是由Master和多个Worker组成,Master负责接收或分配任务,Worker负责具体任务的执行.Master将在所有Worker完成之后接管处理结果再行处理    


### 监听线程执行状态

```scala
object MasterWorkerDemoApp {
  def main(args: Array[String]): Unit = {
    val data = List(1,2,3,4,5,6,7,8,9,10)
    val master = new Thread(new Master(data))
    master.start()
  }
}


class Worker(data: List[Int], state: WorkerState) extends Runnable {
  override def run(): Unit = {
    data.foreach(item => {
      state.result += item
      Thread.sleep(100);
    })
    state.isCompleted = true
  }
}

case class WorkerState(var isCompleted: Boolean, var result: Int)

class Master(datas: List[Int]) extends Runnable {
  override def run(): Unit = {
    val dataSplitTupple = datas.partition(x => x % 3 == 0)
    val dataSplits = List(dataSplitTupple._1, dataSplitTupple._2);

    val dataSplitWorker = dataSplits.map(dataSplit => {
      val workerState = WorkerState(false, 0)
      (new Worker(dataSplit, workerState), workerState)
    })

    dataSplitWorker.foreach(x => new Thread(x._1).start())

    var isAllComplet = false
    while (isAllComplet != true) {
      isAllComplet = dataSplitWorker.filter(x => !x._2.isCompleted).length <= 0
      if (isAllComplet) {
        val sum = dataSplitWorker.map(x => x._2.result).sum
        println(sum)
      }
    }
  }
}
```

### CyclicBarrier锁  

```scala
object MasterWorkerDemoApp {
  def main(args: Array[String]): Unit = {
    val data = List(1,2,3,4,5,6,7,8,9,10)
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
      try{
        cb.await()
      }
      catch {
        // CyclicBarrier 这两个异常是值得手动捕获处理的
        case e:InterruptedException => //本线程在等待过程中被中断抛出,Demo就直接退出好了,所以吃掉异常
        case e:BrokenBarrierException => // CyclicBarrier特有异常,线程等待中被Reset,CyclicBarrier被损坏,或是被中断都会抛出
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

    try{
      cb.await()
    }
    catch {
      // CyclicBarrier 这两个异常是值得手动捕获处理的
      case e:InterruptedException => //本线程在等待过程中被中断抛出,Demo就直接退出好了,所以吃掉异常
      case e:BrokenBarrierException => // CyclicBarrier特有异常,线程等待中被Reset,CyclicBarrier被损坏,或是被中断都会抛出
    }

    val sum = dataSplitWorker.map(x => x._2.result).sum
    println(sum)
  }
}
```