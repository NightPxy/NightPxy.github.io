---
layout: post
title:  "java-ArrayBlockingQueue"
date:   2018-07-07 13:31:01 +0800
categories: java
tag: [java,多线程]
---

* content
{:toc}

## 概述    

ArrayBlockingQueue 是以数组为载体实现的阻塞队列  
数组本质上是下标操作.通过引入 入队和出队 两个下标,就可以技巧性的将数组变为一个队列实现  

ArrayBlockingQueue 是阻塞队列的一个实现形式  
* 试图在队列满时入队可以阻塞生产者,直到队列有新的空间容纳元素  
* 试图在队列空时出队可以阻塞消费者,直到队列有元素可供消费  
*对于阻塞队列而言,ArrayBlockingQueue是一个有界阻塞(有队列上限,且不能扩容)  

ArrayBlockingQueue 使用ReentranLock,以及两个Condition notEmpty和notFull完成并发控制  
* ReentranLock 完成整个队列操作的线程安全  
* lock.notEmpty 完成对消费者线程的等待阻塞  
* lock.notFull 完成对生产者的等待阻塞  

## Demo  

```scala
object ProducorConsumerDemoApp {
  def main(args: Array[String]): Unit = {

    val queue = new ArrayBlockingQueue[String](2)
    val threads = List(
      new Thread(new Producor("p1","1", queue)),
      new Thread(new Producor("p2","2", queue)),
      new Thread(new Consumer("c1", queue)),
      new Thread(new Consumer("c2", queue)),
      new Thread(new Consumer("c3", queue))
    )

    threads.foreach(x => x.start())
    println("开始执行")

  }
}

class Producor(name:String, info: String, queue: ArrayBlockingQueue[String]) extends Runnable {
  override def run(): Unit = {
    while(true){
      Thread.sleep(1000)
      println(s" Producor-$name working")
      queue.put(info)
    }
  }
}

class Consumer(name:String,queue: ArrayBlockingQueue[String]) extends Runnable {
  override def run(): Unit = {
    while (true){
      Thread.sleep(300)
      println(s" Consumer-$name working")
      val info =  queue.take()
      if(info != null){
        println(info)
      }
    }
  }
}
```

## 源码解析  

### Array的队列实现  


```java
//Array的特征是下标操作,为了实现队列的进,出就设置进出两个下标  
int takeIndex;
int putIndex;
int count;

private void enqueue(E x) {
	final Object[] items = this.items;
	//入队始终加入到入队下标中,如果入队下标满就从头开始
	//值得学习的数组下标循环小技巧  
	items[putIndex] = x;
	if (++putIndex == items.length)
		putIndex = 0;
	count++;
	//有入队就通知所有等待消费的线程解除阻塞(如果有的话)  
	notEmpty.signal();
}
    
private E dequeue() {  
	final Object[] items = this.items;  
	@SuppressWarnings("unchecked")  
	E x = (E) items[takeIndex];  
	items[takeIndex] = null;  
	//出队始终从出队下标中出,如果入队下标满就从头开始  
	if (++takeIndex == items.length)  
		takeIndex = 0;  
	count--;//当前拥有元素个数减1  
	if (itrs != null)  
		itrs.elementDequeued();  
	//有出队就通知所有等待生产的线程解除阻塞(如果有的话)
	notFull.signal();
	return x;  
}  
```

### 入队  

ArrayBlockingQueue 入队有add,offer,put方法  
* add 只是接口要求,实质等于offer方法  
* offer(默认情况) 会在队列满时返回false,而不会阻塞当前线程  
* put 会在队列满时阻塞线程直到可以入队为止  

```java
public boolean offer(E e) {
	checkNotNull(e);
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		if (count == items.length)
			return false;
		else {
			enqueue(e);
			return true;
		}
	} finally {
		lock.unlock();
	}
}
public void put(E e) throws InterruptedException {
	checkNotNull(e);
	//在局部作用域内final接收一次变量(一个非常有技巧性的写法(值得学习))
	//好处有 1.栈帧内缓存而不用从主内存读取 2.利用final隐式的保持可见性和禁止指令重排  
	final ReentrantLock lock = this.lock;
	//上锁 与lock区别在于lockInterruptibly阻塞中允许异常抛出
	lock.lockInterruptibly();
	try {
	    //这里使用while是一个相当有技巧的写法(值得学习)
	    //因为解除阻塞后是等待写入者们新一轮竞争,所以会在解除阻塞后再次判断确实有效
		while (count == items.length)
			notFull.await();
		enqueue(e);
	} finally {
		lock.unlock();
	}
}
public boolean offer(E e, long timeout, TimeUnit unit)
	throws InterruptedException {

	checkNotNull(e);
	long nanos = unit.toNanos(timeout);
	final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		while (count == items.length) {
			if (nanos <= 0)
				return false;
			nanos = notFull.awaitNanos(nanos);
		}
		enqueue(e);
		return true;
	} finally {
		lock.unlock();
	}
}
```

### 出队  

ArrayBlockingQueue 出队也有poll,take,remove方法   
* remove 是接口要求,实质是poll方法  
* poll不会阻塞消费线程,将在队列为空时返回null  
* take会阻塞消费线程,直到队列有元素可以返回为止  

```java
public E poll() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		return (count == 0) ? null : dequeue();
	} finally {
		lock.unlock();
	}
}

public E take() throws InterruptedException {
	final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		while (count == 0)
			notEmpty.await();
		return dequeue();
	} finally {
		lock.unlock();
	}
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
	long nanos = unit.toNanos(timeout);
	final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		while (count == 0) {
			if (nanos <= 0)
				return null;
			nanos = notEmpty.awaitNanos(nanos);
		}
		return dequeue();
	} finally {
		lock.unlock();
	}
}

//真正的出队方法
private E dequeue() {  
	final Object[] items = this.items;  
	@SuppressWarnings("unchecked")  
	E x = (E) items[takeIndex];  
	items[takeIndex] = null;  
	if (++takeIndex == items.length)  
		takeIndex = 0;  
	count--;//当前拥有元素个数减1  
	if (itrs != null)  
		itrs.elementDequeued();  
	notFull.signal();//有一个元素取出成功，那肯定队列不满  
	return x;  
}  
```

### 队列管理  

```java

public boolean remove(Object o) {
	if (o == null) return false;
	final Object[] items = this.items;
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		if (count > 0) {
			final int putIndex = this.putIndex;
			int i = takeIndex;
			do {
				if (o.equals(items[i])) {
					removeAt(i);
					return true;
				}
				if (++i == items.length)
					i = 0;
			} while (i != putIndex);
		}
		return false;
	} finally {
		lock.unlock();
	}
}

public boolean contains(Object o) {
	if (o == null) return false;
	final Object[] items = this.items;
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		if (count > 0) {
			final int putIndex = this.putIndex;
			int i = takeIndex;
			do {
				if (o.equals(items[i]))
					return true;
				if (++i == items.length)
					i = 0;
			} while (i != putIndex);
		}
		return false;
	} finally {
		lock.unlock();
	}
}

public E peek() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		return itemAt(takeIndex); // null when queue is empty
	} finally {
		lock.unlock();
	}
}

 public int size() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		return count;
	} finally {
		lock.unlock();
	}
}
```