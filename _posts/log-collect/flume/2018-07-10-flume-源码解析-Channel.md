---
layout: post
title:  "flume-源码解析-Channel"
date:   2018-07-9 13:31:01 +0800
categories: flume
tag: [flume,源码解析]
---

* content
{:toc}

# 版本

基于 Flume CDH 版  
版本号: flume-ng-1.6.0-cdh5.7.0  

# 核心流程  

## ChannelProcessor  

ChannelProcessor 公开将 Event 放入到 Channel 的所有操作  

## BasicChannelSemantics  

BasicChannelSemantics 是定义Channel的抽象类接口.  
它定义了 Channel 所应有的操作方法.  
它的核心方法两个 put take,分别对应Channel的两大核心操作:存入数据 取出数据  

* 事务性
在基类中这两大操作本身是事务性的,在基类中说明这种事务性是涵盖所有的Channel的,具体稍后讨论 
* Channel 的操作,实质是桥接的 transaction 的操作  
```java
public void put(Event event) throws ChannelException {
    BasicTransactionSemantics transaction = currentTransaction.get();
    Preconditions.checkState(transaction != null,
        "No transaction exists for this thread");
    transaction.put(event);
  }

public Event take() throws ChannelException {
    BasicTransactionSemantics transaction = currentTransaction.get();
    Preconditions.checkState(transaction != null,
        "No transaction exists for this thread");
    return transaction.take();
  }
```
事务的通用手法:二次提交  
数据放入临时缓冲区putlist(事务区),最后由commit检查是否已有足够的缓冲区到Channel.有则二次提交事务区内数据到Channel,没有则回滚(清空)事务区

# 核心方法

以 **MemoryChannel** 为例  



## Transaction

从 BasicChannelSemantics 知道,Channel 操作的实质是桥接的 Transaction 操作  

实现抽象类BasicTransactionSemantics  
提供对put take 的事务性操作  

MemoryChannel 中, 定义了内部类 MemoryTransaction,实质完成了 MemoryChannel 的操作  

### 核心属性

* LinkedBlockingDeque<Event> takeList  
Take操作的临时缓冲区  
* LinkedBlockingDeque<Event> putList  
Put操作的临时缓冲区  
* final ChannelCounter channelCounter  
用于记录 channel 当前 event 数量的线程安全计数器   
* putByteCounter 
putList的eventByteSize总计  
* takeByteCounter
takeList的eventByteSize总计  


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
//真正提交事务区数据到Channel putlist放入queue再清空takelist
//对于 take.commit Sink已经调用take取出数据.(取出数据留存takelist)
//  此时的commit 代表Sink确认success 就是简单的清空缓冲区
//对于 put.commit Source已经调用put将数据全部写入putlist
//  此时的commit就是确保 从putlist写到channel.queue
protected void doCommit() throws InterruptedException {
      /**
      * putlist放入queue再清空takelist 正常的处理逻辑是 
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