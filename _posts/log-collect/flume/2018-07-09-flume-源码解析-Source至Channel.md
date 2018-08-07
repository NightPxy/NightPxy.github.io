---
layout: post
title:  "flume-源码解析-Source至Channel"
date:   2018-07-09 13:31:01 +0800
categories: collect
tag: [flume,源码解析]
---

* content
{:toc}


## Source -> Channel   


### ChannelSelector  

org.apache.flume.ChannelSelector

### ChannelProcessor  

org.apache.flume.channel.ChannelProcessor
Flume 由Source采集数据,但是Source采集到数据,最终是需要交给Channel  
这个由Source到Channel的过程  就是由 ChannelProcessor 负责处理.具体如下  
* 负责对一个或多个Channel数据提交的抽象实现  
* 负责提交触发器的执行  
* 负责事务过程处理,包括处理提交,回滚等  
* 负责Channel选择器策略的执行等等   

**核心逻辑**

```java
/**
* 处理批量提交事务 (还有个处理单Event的processEvent方法,大致相同)
*/
public void processEventBatch(List<Event> events) {
    Preconditions.checkNotNull(events, "Event list must not be null");

    //触发器的执行
    events = interceptorChain.intercept(events);

    //必须通道和可选通道的读取暂存数据块(每个通道独立一个)
    //键是目标通道实例,值是需要写入该通道的eventslist
    Map<Channel, List<Event>> reqChannelQueue =
        new LinkedHashMap<Channel, List<Event>>();

    Map<Channel, List<Event>> optChannelQueue =
        new LinkedHashMap<Channel, List<Event>>();

    //遍历 events,写入每个通道的暂存数据块eventQueue
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

    // 处理必须通道的数据提交
    // 必须通道是同事务提交,其中中出现任意问题都会回滚全部通道
    // 详见 executeChannelTransaction 事务提交方法
    for (Channel reqChannel : reqChannelQueue.keySet()) {
      List<Event> batch = reqChannelQueue.get(reqChannel);
      executeChannelTransaction(reqChannel, batch, false);
    }

    // 处理可选通道的数据提交
    // 可选通道是独立事务提交,即出现任何问题会回滚该通道数据,但不会影响其它通道
    // OptionalChannelTransactionRunnable 可选通道的数据提交还是以异步线程的形式
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
