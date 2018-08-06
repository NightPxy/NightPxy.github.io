---
layout: post
title:  "flume-源码解析-Source至Channel"
date:   2018-07-09 13:31:01 +0800
categories: flume
tag: [flume,源码解析]
---

* content
{:toc}


## Source -> Channel   



### ChannelSelector  

org.apache.flume.ChannelSelector

### ChannelProcessor  

org.apache.flume.channel.ChannelProcessor

从一个Source提交数据到Channel的处理实质是由 ChannelProcessor 完成   

它依赖于org.apache.flume.ChannelSelector,代表该Source下涵盖哪些Channel  
它负责Source-Events -> Channels的骨干逻辑,比如拦截器执行,Channel批量事务等    

**核心逻辑**

```java
/**
* 处理批量提交事务 (还有个处理单Event的processEvent方法,大致相同)
*/
public void processEventBatch(List<Event> events) {
    Preconditions.checkNotNull(events, "Event list must not be null");

    //触发器的执行
    events = interceptorChain.intercept(events);

    //必须队列和可选队列的读取暂存数据块(每个channel独立一个)
    //键是目标channel实例,值是需要写入该channel的events
    Map<Channel, List<Event>> reqChannelQueue =
        new LinkedHashMap<Channel, List<Event>>();

    Map<Channel, List<Event>> optChannelQueue =
        new LinkedHashMap<Channel, List<Event>>();

    //遍历 events,写入每个channel的暂存数据块eventQueue
    for (Event event : events) {
      List<Channel> reqChannels = selector.getRequiredChannels(event);
      for (Channel ch : reqChannels) {
        List<Event> eventQueue = reqChannelQueue.get(ch);
        if (eventQueue == null) {
          eventQueue = new ArrayList<Event>();
          reqChannelQueue.put(ch, eventQueue);
        }
        eventQueue.add(event);
      }

      List<Channel> optChannels = selector.getOptionalChannels(event);
      for (Channel ch: optChannels) {
        List<Event> eventQueue = optChannelQueue.get(ch);
        if (eventQueue == null) {
          eventQueue = new ArrayList<Event>();
          optChannelQueue.put(ch, eventQueue);
        }
        eventQueue.add(event);
      }
    }

    // 处理必须队列的数据提交
    // 必须队列是同事务提交,必须队列中出现任意问题都会回滚全部必须队列
    // executeChannelTransaction 事务提交方法见下
    for (Channel reqChannel : reqChannelQueue.keySet()) {
      List<Event> batch = reqChannelQueue.get(reqChannel);
      executeChannelTransaction(reqChannel, batch, false);
    }

    // 处理可选队列的数据提交
    // 可选队列是独立事务提交,即出现任何问题会回滚该队列数据,但不会影响其它队列
    // OptionalChannelTransactionRunnable 可选队列的数据提交还是以异步线程的形式
    for (Channel optChannel : optChannelQueue.keySet()) {
      List<Event> batch = optChannelQueue.get(optChannel);
      execService.submit(new OptionalChannelTransactionRunnable(optChannel, batch));
    }
  }

/**
* 事务提交 eventBatch 到 channel
* channel基类提供开启,提交,回滚事务的抽象实现.
* 所以在ChannelProcessor,不关心事务的具体实现细节,按照规定流程走就是了
* 
* 数据提交数据到chanel的核心操作就是 channel.put 这个方法后面详述
*/
private static void executeChannelTransaction(Channel channel, List<Event> batch, boolean isOptional) {
    Transaction tx = channel.getTransaction();
    Preconditions.checkNotNull(tx, "Transaction object must not be null");
    try {
      tx.begin();

      for (Event event : batch) {
        channel.put(event);
      }

      tx.commit();
    } catch (Throwable t) {
      tx.rollback();
      //如果是系统级异常,就会抛出异常,此时会影响所有channels
      if (t instanceof Error) {
        LOG.error("Error while writing to channel: " +
                channel, t);
        throw (Error) t;
      } 
      //不是系统级异常
      //   如果是可选channel就吃掉异常,只回滚本通道数据
      //   如果是必须channel就抛出异常,此时会回滚所有channels
      else if(!isOptional) {
          throw new ChannelException("Unable to put batch on required " +
                  "channel: " + channel, t);
      }
    } finally {
      tx.close();
    }
  }
```
