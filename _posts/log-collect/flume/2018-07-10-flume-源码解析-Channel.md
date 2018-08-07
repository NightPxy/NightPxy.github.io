---
layout: post
title:  "flume-源码解析-Channel"
date:   2018-07-10 13:31:01 +0800
categories: collect
tag: [flume,源码解析]
---

* content
{:toc}


## 概述

由Flume架构得知,一个Channel需要实现Channel BasicTransactionSemantics


## MemoryChannel 的分析  

## MemoryChannel 主类  

**类声明**  

由前章Flume源码架构得知,一个Channel是必须实现自 org.apache.flume.Channel  
在MemoryChannel类定义就是继承 BasicChannelSemantics  

前文已经说过,这里本质上是组合模式设计,将Channel的逻辑整体拆分 通道自身维护和通道事务的 put/get 操作两大部分

所以 MemoryChannel 主类就是负责通道自身维护以及如何产生一个事务块,具体如下  
* 产生一个事务块  
* 通道的Event数据容器  
* 通道并发锁  
* 内存资源锁(Memory通道特有)  

```java
public class MemoryChannel extends BasicChannelSemantics {
    //通道数据载体,在MemoryChannel就是存在内存中的LinkedBlockingDeque<Event>
    private LinkedBlockingDeque<Event> queue;
    //通道的并发锁,  
    private Object queueLock = new Object();
    //通道的内存剩余资源锁,这里使用的是一个Semaphore
    private Semaphore queueRemaining;
    //通道的内存已占用资源锁,....
    private Semaphore queueStored;
    //通道的Event数量计数器
    private ChannelCounter channelCounter;
    
    .....
    
    //在MemoryChannel定义一个内部类负责通道事务的 put/get 操作 下面详述
    private class MemoryTransaction extends BasicTransactionSemantics {
        ...
    }
}
```

## MemoryTransaction

在MemoryChannel中,定义了通道自身的维护,包括数据容器,锁等   
在MemoryTransaction中,定义了通道的put/take事务操作过程  

### 事务的一般处理手段   

处理事务的一般处理手段,就是二次提交思路  大体步骤为  
* 事务块数据第一次提交至暂存区  
* 全部处理成功后,由暂存区二次提交至目标区,如果中途失败,将回滚暂存区  

在Flume的事务处理中,也是基于的这种思路  

### 核心属性

* LinkedBlockingDeque<Event> takeList  
Take操作的临时事务区  
* LinkedBlockingDeque<Event> putList  
Put操作的临时事务区  
* final ChannelCounter channelCounter  
用于记录 channel 当前 event 数量的线程安全计数器   
* putByteCounter 
putList的eventByteSize总计  
* takeByteCounter
takeList的eventByteSize总计  

### Tran的两条调用线  

就是由 BasicTransactionSemantics 定义要求用户实现的4个方法  
* doPut  
* doTake   
* doCommit   
* doRollback  

由架构章节得知,Channel是整个Flume的中心地位.逻辑都是围绕Channel进行展开的,所以Channel的这四个方法,其实是分作两条调用线的  
* doPut -> doCommit/doRollback 
这是Source到Channel的调用线.
Source读取到数据,然后channel.put(doput)提交到缓冲区   
完毕后commit(缓冲区数据到Channel)或rollback(清空缓冲区)  
* doTake -> doCommit/doRollback 
这是Channel到Sink的调用线  
有Sink调用channel.take拉取数据,每一个从take拉取的数据都会进入缓冲区  
Sink输出完毕后,commit(清空缓冲区)或rollback(缓冲区数据重新回到Channel)

详细描述如下  

### void  doPut(Event event)  

```java
protected void doPut(Event event) throws InterruptedException {
      //每进入一个Event,channel的event数量计数器put(加一)
      channelCounter.incrementEventPutAttemptCount();
      //计算进入event的eventByteSize(只针对event的body)
      int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);
      //LinkedBlockingDeque.offer 如果立即可行且不违反容量限制，则插入队列尾部,成功返回true    
      if (!putList.offer(event)) {
        throw new ChannelException(
          "Put queue for MemoryTransaction of capacity " +
            putList.size() + " full, consider committing more frequently, " +
            "increasing capacity or increasing thread count");
      }
      //putList的eventByteSize 增加
      putByteCounter += eventByteSize;
    }
```

### Event doTake()  

```java
 protected Event doTake() throws InterruptedException {
      //每取走一个Event,channel的event数量计数器take(减一)
      channelCounter.incrementEventTakeAttemptCount();
      if(takeList.remainingCapacity() == 0) {
        throw new ChannelException("Take list for MemoryTransaction, capacity " +
            takeList.size() + " full, consider committing more frequently, " +
            "increasing capacity, or increasing thread count");
      }
      
      // LinkedBlockingDeque.remainingCapacity 
      // 尝试获取queue数量的许可，如果没有则代表没有数据可以取，直接返回
      if(!queueStored.tryAcquire(keepAlive, TimeUnit.SECONDS)) {
        return null;
      }
      Event event;
      /**
      *锁channel.queue出队头event放入take队列
      *同时叠加该event的eventByteSize到take队列
      */
      synchronized(queueLock) {
        event = queue.poll();
      }
      Preconditions.checkNotNull(event, "Queue.poll returned NULL despite semaphore " +
          "signalling existence of entry");
      takeList.put(event);

      int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);
      takeByteCounter += eventByteSize;

      return event;
    }
```

### void doCommit()  

```java
//对于 put.commit Source已经调用put将数据全部写入putlist
//  此时的commit就是确保 从putlist写到channel.queue
//对于 take.commit Sink已经调用take取出数据.(取出数据留存takelist)
//  此时的commit 代表Sink确认success 就是简单的清空缓冲区
protected void doCommit() throws InterruptedException {
      /**
      * 对于 put.commit 正常的处理逻辑是 
      * 首先回收takelist.size 然后再申请 putlist.size 
      * 这里它用了一个技巧性的写法 让回收和申请合并为一步操作
      * 这个技巧性写法就是比较 申请(putlist.size)与释放(takelist.size)的容量查
      * 如果 申请大于释放量 则只需要尝试申请容量差值就可以了
      * 如果 申请小于释放量 则无需去尝试申请.因为释放后的剩余必然可以容纳申请量
      * 这样做的好处是可以减少锁块范围 否则必须将两步操作都纳入锁范围
      */
      int remainingChange = takeList.size() - putList.size();
      if(remainingChange < 0) {
        if(!bytesRemaining.tryAcquire(putByteCounter, keepAlive,
          TimeUnit.SECONDS)) {
          throw new ChannelException("Cannot commit transaction. Byte capacity " +
            "allocated to store event body " + byteCapacity * byteCapacitySlotSize +
            "reached. Please increase heap space/byte capacity allocated to " +
            "the channel as the sinks may not be keeping up with the sources");
        }
        if(!queueRemaining.tryAcquire(-remainingChange, keepAlive, TimeUnit.SECONDS)) {
          bytesRemaining.release(putByteCounter);
          throw new ChannelFullException("Space for commit to queue couldn't be acquired." +
              " Sinks are likely not keeping up with sources, or the buffer size is too tight");
        }
      }
      int puts = putList.size();
      int takes = takeList.size();
      
      synchronized(queueLock) {
        // channel.queueLock锁下 将putlist全部放入queue
        // 如果有任何放入queue失败,都会抛出异常,写入失败
        // 注意异常内容: this shouldn't be able to happen
        // 在这套事务体系中,这种错误是不允许发生的
        //   因为事务回滚不了queue,所以对部分进入queue的情况是处理不了的
        // 实际过程也不太可能出现写入queue异常
        //   这是一个由JDK提供的不掺杂任何业务的写入,且前面已经检测过容量,基本没有失败的可能
        if(puts > 0 ) {
          while(!putList.isEmpty()) {
            if(!queue.offer(putList.removeFirst())) {
              throw new RuntimeException("Queue add failed, this shouldn't be able to happen");
            }
          }
        }
        //事务块(putlist写入queue)完成,清空两个缓冲区
        putList.clear();
        takeList.clear();
      }
      //channel的内存控制信号量bytesRemaining释放takelist.byteSize
      bytesRemaining.release(takeByteCounter);
      //清空两个缓冲区的byteSize的计数器(个人觉得这里两句应该放入锁块中,嘻嘻)
      takeByteCounter = 0;
      putByteCounter = 0;

      //channel的已使用队列容量释放 putList.size
      queueStored.release(puts);
      
      // channel的未使用队列容量释放 (takeList.size - putList.size)差值
      if(remainingChange > 0) {
        queueRemaining.release(remainingChange);
      }
      // 如果是commit put,通知put
      if (puts > 0) {
        channelCounter.addToEventPutSuccessCount(puts);
      }
      // 如果是commit take,通知take
      if (takes > 0) {
        channelCounter.addToEventTakeSuccessCount(takes);
      }
      // 最后标记下 commit 之后当前 channel.queue 的event数量
      channelCounter.setChannelSize(queue.size());
    }
```

### void doRollback()

```java
//rollback逻辑与commit刚好相反
//如果take rollback 表示Sink放弃这次读取 
//  则takelist数据全部重新回到channel.queue
//如果put rollback 表示Source放弃这次写入
//  这个处理简单 清空putlist就可以了
protected void doRollback() {
      int takes = takeList.size();
      synchronized(queueLock) {
        //锁下将takelist数据重新回滚到channel.queue中
        Preconditions.checkState(queue.remainingCapacity() >= takeList.size(), "Not enough space in memory channel " +
            "queue to rollback takes. This should never happen, please report");
        while(!takeList.isEmpty()) {
          queue.addFirst(takeList.removeLast());
        }
        putList.clear();
      }
      bytesRemaining.release(putByteCounter);
      putByteCounter = 0;
      takeByteCounter = 0;

      queueStored.release(takes);
      channelCounter.setChannelSize(queue.size());
    }
```