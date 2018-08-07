---
layout: post
title:  "flume-源码解析-Sink"
date:   2018-07-11 13:31:01 +0800
categories: collect
tag: [flume,源码解析]
---

* content
{:toc}


## Channel  -> Sink 

### SinkProcessor

org.apache.flume.SinkProcessor,它代表以怎样的策略处理Channel到Sink的过程 ,其内部包裹一个或多个Sink   

怎样的输出策略是指,SinkProcessor有三种不同的分类,分别对应三种不同的执行输出策略  
* 无策略(此时Sink只能有一个)  
* 容错策略  
* 负载均衡策略  

```java
public interface SinkProcessor extends LifecycleAware, Configurable {
  Status process() throws EventDeliveryException;
  void setSinks(List<Sink> sinks);
}
```

### DefaultSinkProcessor  

org.apache.flume.sink.DefaultSinkProcessor 是一种默认策略(无)输出  
在 DefaultSinkProcessor 中,Sink有且仅有一个,也仅仅只是调用 sink.process  

```java
public Status process() throws EventDeliveryException {
    return sink.process();
  }
```

### FailoverSinkProcessor   

org.apache.flume.sink.FailoverSinkProcessor 是一种 容错输出.  
这种容错输出中,Sink可能会有多个,将会顺序尝试,有一个sink.process成功,输出结束

在FailoverSinkProcessor中两个池,  activeSink 和 failedSinks  
* 同一时间只有一个 activeSink,如果可用,会一直使用该Sink输出  
* 每一个发现失败的Sink都会仍进 failedSinks,并设置一个冷却器,冷却后会重新进行尝试输出  

```java
public Status process() throws EventDeliveryException {
    // Retry any failed sinks that have gone through their "cooldown" period
    Long now = System.currentTimeMillis();
    
    // 刷新failedSinks 如果存在已经过了冷却时间的failedSink,就取出尝试是否可用
    while(!failedSinks.isEmpty() && failedSinks.peek().getRefresh() < now) {
      FailedSink cur = failedSinks.poll();
      Status s;
      try {
        //尝试是否用就是调用目标sink.process,成功即表示可用
        s = cur.getSink().process();
        if (s  == Status.READY) {
          //尝试使用成功,activeSink切换为该sink
          liveSinks.put(cur.getPriority(), cur.getSink());
          activeSink = liveSinks.get(liveSinks.lastKey());
          logger.debug("Sink {} was recovered from the fail list",
                  cur.getSink().getName());
        } else {
          //尝试使用失败.该sink继续放回failedSinks
          failedSinks.add(cur);
        }
        return s;
      } catch (Exception e) {
        //使用出现异常,依然视为失败,放回failedSinks
        cur.incFails();
        failedSinks.add(cur);
      }
    }

    Status ret = null;
    //如果出现一个可用的activeSink,就会活锁连续使用该sink
    while(activeSink != null) {
      try {
        ret = activeSink.process();
        return ret;
      } catch (Exception e) {
        logger.warn("Sink {} failed and has been sent to failover list",
                activeSink.getName(), e);
        //如果 activeSink 使用失败,就移除active并切换尝试下一个
        activeSink = moveActiveToDeadAndGetNext();
      }
    }

    throw new EventDeliveryException("All sinks failed to process, " +
        "nothing left to failover to");
  }
```

### LoadBalancingSinkProcessor

org.apache.flume.sink.LoadBalancingSinkProcessor 表示负载均衡Sink  
内部使用了 顺序使用 随机使用 两种负载均衡Sink选择策略

```java
public Status process() throws EventDeliveryException {
    Status status = null;
    //createSinkIterator依据当前SinkList构造一个Sink顺序器,策略如下:
    //  RoundRobinSinkSelector 将固定顺序组织 SinkList
    //  RandomOrderSinkSelector 将随机顺序组织 SinkList
    Iterator<Sink> sinkIterator = selector.createSinkIterator();
    //根据策略顺序 挨个使用Sink 有任意成功即结束使用
    while (sinkIterator.hasNext()) {
      Sink sink = sinkIterator.next();
      try {
        status = sink.process();
        break;
      } catch (Exception ex) {
        selector.informSinkFailed(sink);
        LOGGER.warn("Sink failed to consume event. "
            + "Attempting next sink if available.", ex);
      }
    }

    if (status == null) {
      throw new EventDeliveryException("All configured sinks have failed");
    }

    return status;
  }
```
