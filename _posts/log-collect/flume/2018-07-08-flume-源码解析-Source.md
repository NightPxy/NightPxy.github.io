---
layout: post
title:  "flume-源码解析-Source"
date:   2018-07-08 13:31:01 +0800
categories: flume
tag: [flume,源码解析]
---

* content
{:toc}


## SpoolDirectorySource  

SpoolDirectorySource 内部通过Start方法启动一个 SpoolDirectoryRunnable 线程来监听读取文件
```java
 Runnable runner = new SpoolDirectoryRunnable(reader, sourceCounter);
 executor.scheduleWithFixedDelay(
        runner, 0, POLL_DELAY_MS, TimeUnit.MILLISECONDS);

super.start();
```

## SpoolDirectoryRunnable  

SpoolDirectoryRunnable 线程 run 核心代码如下

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
            //ChannelProcessor其实就是代指的Channel,这是Flume对一组Channels的抽象封装.
            //ChannelProcessor.processEventBatch 将数据写到每一个channel中
            // 写入channel,就是调用channel的put方法写入到putlist提交事务块中
            getChannelProcessor().processEventBatch(events);
            //reader 尝试提交
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