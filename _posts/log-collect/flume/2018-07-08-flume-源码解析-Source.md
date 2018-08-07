---
layout: post
title:  "flume-源码解析-Source"
date:   2018-07-08 13:31:01 +0800
categories: collect
tag: [flume,源码解析]
---

* content
{:toc}


## 概述

由Flume架构得知,一个Source实现基础Source
这里以SpoolDirectorySource为例 详述一个Source子类的具体实现过程

## SpoolDirectorySource 的分析  

## SpoolDirectorySource 主类  

**类声明**  

```java
public class SpoolDirectorySource 
  extends AbstractSource 
  implements Configurable, EventDrivenSource {
```

首先关注下SpoolDirectorySource类声明  
* 继承自AbstractSource  
AbstractSource 是一个继承自Source的再实现抽象类,它是为Source提供默认的支持  

* 实现 EventDrivenSource接口  
这是为 SpoolDirectorySource 打上一个类型标记,这代表它是一个无需驱动参与可直接读取的Source

**核心处理**

Spool 已经标记为EventDrivenSource,这代表它将是以EventDrivenSourceRunner来执行  
在这种策略下,Flume将启动Source自身线程执行,相当于每一个Source实例一个执行线程  

所以在内部需要通过Start方法启动一个 SpoolDirectoryRunnable 线程来监听读取文件  

```java
 Runnable runner = new SpoolDirectoryRunnable(reader, sourceCounter);
 executor.scheduleWithFixedDelay(
        runner, 0, POLL_DELAY_MS, TimeUnit.MILLISECONDS);

super.start();
```

## SpoolDirectoryRunnable  

SpoolDirectoryRunnable 是 SpoolDirectorySource 的一个线程内部类   
run 这里就是如何读取数据写入Channel了  核心代码如下

```java
public void run() {
      int backoffInterval = 250;
      try {
        while (!Thread.interrupted()) {
          // 通过 reader 读取器尝试从监听目录中读取 batchSize 个 Event
          List<Event> events = reader.readEvents(batchSize);
          if (events.isEmpty()) {
            break;
          }
          sourceCounter.addToEventReceivedCount(events.size());
          sourceCounter.incrementAppendBatchReceivedCount();

          try {
            //一旦读取到Event,就会用 getChannelProcessor来得到ChannelProcessor
            //ChannelProcessor 是对从Source写数据到Channel的过程封装
            //这里简单理解为对一组Channels
            //ChannelProcessor.processEventBatch 将一批数据写到每一个channel中
            getChannelProcessor().processEventBatch(events);
            //提交 注意这里的提交不是事务的提交,事务提交包裹在ChannelProcessor中
            //这里的提交是 Spool 读取完毕后比较文件已读的过程
            reader.commit();
          } catch (ChannelException ex) {
            logger.warn("The channel is full, and cannot write data now. The " +
              "source will try again after " + String.valueOf(backoffInterval) +
              " milliseconds");
            hitChannelException = true;
            if (backoff) {
              TimeUnit.MILLISECONDS.sleep(backoffInterval);
              backoffInterval = backoffInterval << 1;
              backoffInterval = backoffInterval >= maxBackoff ? maxBackoff :
                                backoffInterval;
            }
            continue;
          }
          backoffInterval = 250;
          sourceCounter.addToEventAcceptedCount(events.size());
          sourceCounter.incrementAppendBatchAcceptedCount();
        }
      } catch (Throwable t) {
        logger.error("FATAL: " + SpoolDirectorySource.this.toString() + ": " +
            "Uncaught exception in SpoolDirectorySource thread. " +
            "Restart or reconfigure Flume to continue processing.", t);
        hasFatalError = true;
        Throwables.propagate(t);
      }
    }
```