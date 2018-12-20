---
layout: post
title:  "线程设计模式-Balking"
date:   2018-07-10 13:31:01 +0800
categories: java
tag: [java]
---

* content
{:toc}



## Balking Pattern  

把一个任务交给一个线程处理,处理警戒如果这个任务已经被它人完成了就直接返回  

Balking Pattern 与 Guarded Suspension 有点类似,都需要一个警戒条件  
不同之处在于警戒不成立时,Guarded选择等待警戒成立.Balking选择直接返回  

Balking Pattern 特别适用诸如与初始化之类的只需要一次执行,但不关心谁在什么时候执行的情况  
Balking Pattern 的核心思路是volatile修饰共享变量,基于volatile可见性保证让多个线程进入时判断自身是否继续执行 

```scala
class Data {
  @volatile var isChanged = false
  var content: String = _

  def change(content: String) = {
    isChanged = true
    this.content = content
  }

  def save(): Unit = {
    if (isChanged == false) return
    println(s"save $content")
  }
}

class SaveThread(data: Data) extends Thread {
  override def run(): Unit = {
    while (true) {
      data.save()
      Thread.sleep(100)
    }
  }
}

object BalkingDemoApp {
  def main(args: Array[String]): Unit = {
    val data = new Data()
    val saveThread = new SaveThread(data);
    val changeThread = new ChangeThread(data);

    saveThread.start()
    changeThread.start()
  }
}
```