---
layout: post
title:  "flume-源码解析-Sink"
date:   2018-07-12 13:31:01 +0800
categories: collect
tag: [flume,源码解析]
---

* content
{:toc}


## 核心流程  

Sink核心流程是 SinkRunner 不断调用 SinkProcessor的Process方法  
* 根据配置的 SinkProcessor,会使用不同的策略来选择 Sink.  
SinkProcessor 有3种,默认DefaultSinkProcessor
* 调用选择 Sink 的 Process 方法  

## HDFSEventSink 分析 

### 类定义  

```java
public class HDFSEventSink extends AbstractSink
```

### Process方法  

由架构章得知,Sink最重要的是process方法,Sink的实际处理就是由SinkRunner不断调用sink.process来完成的

**HDFSEventSink** 的 *Process* 源码如下

```java
/**
   * Pull events out of channel and send it to HDFS. Take at most batchSize
   * events per Transaction. Find the corresponding bucket for the event.
   * Ensure the file is open. Serialize the data and write it to the file on
   * HDFS. <br/>
   * This method is not thread safe.
   */
  public Status process() throws EventDeliveryException {
    Channel channel = getChannel();
    //每一次process过程,就是一个完整的事务块
    Transaction transaction = channel.getTransaction();
    List<BucketWriter> writers = Lists.newArrayList();
    transaction.begin();
    try {
      int txnEventCount = 0;
      for (txnEventCount = 0; txnEventCount < batchSize; txnEventCount++) {
        Event event = channel.take();
        if (event == null) {
          break;
        }

        // 计算出写到HDFS的两个关键属性 filepath 和 filename
        String realPath = BucketPath.escapeString(filePath, event.getHeaders(),
            timeZone, needRounding, roundUnit, roundValue, useLocalTime);
        String realName = BucketPath.escapeString(fileName, event.getHeaders(),
          timeZone, needRounding, roundUnit, roundValue, useLocalTime);
        //计算出写入HDFS的绝对路径
        //DIRECTORY_DELIMITER 其实System.getProperty("file.separator")
        //取出操作系统的默认文件分割符
        String lookupPath = realPath + DIRECTORY_DELIMITER + realName;
        
        
        BucketWriter bucketWriter;
        HDFSWriter hdfsWriter = null;
        // Callback to remove the reference to the bucket writer from the
        // sfWriters map so that all buffers used by the HDFS file
        // handles are garbage collected.
        WriterCallback closeCallback = new WriterCallback() {
          @Override
          public void run(String bucketPath) {
            LOG.info("Writer callback called.");
            synchronized (sfWritersLock) {
              sfWriters.remove(bucketPath);
            }
          }
        };
        synchronized (sfWritersLock) {
          bucketWriter = sfWriters.get(lookupPath);
          // we haven't seen this file yet, so open it and cache the handle
          if (bucketWriter == null) {
            //fileType: SequenceFile、DataStream、CompressedStream 三种HdfsWriter
            hdfsWriter = writerFactory.getWriter(fileType);
            bucketWriter = initializeBucketWriter(realPath, realName,
              lookupPath, hdfsWriter, closeCallback);
            sfWriters.put(lookupPath, bucketWriter);
          }
        }

        // track the buckets getting written in this transaction
        if (!writers.contains(bucketWriter)) {
          writers.add(bucketWriter);
        }

        // Write the data to HDFS
        try {
          bucketWriter.append(event);
        } catch (BucketClosedException ex) {
          LOG.info("Bucket was closed while trying to append, " +
            "reinitializing bucket and writing event.");
          hdfsWriter = writerFactory.getWriter(fileType);
          bucketWriter = initializeBucketWriter(realPath, realName,
            lookupPath, hdfsWriter, closeCallback);
          synchronized (sfWritersLock) {
            sfWriters.put(lookupPath, bucketWriter);
          }
          bucketWriter.append(event);
        }
      }

      if (txnEventCount == 0) {
        sinkCounter.incrementBatchEmptyCount();
      } else if (txnEventCount == batchSize) {
        sinkCounter.incrementBatchCompleteCount();
      } else {
        sinkCounter.incrementBatchUnderflowCount();
      }

      // flush all pending buckets before committing the transaction
      for (BucketWriter bucketWriter : writers) {
        bucketWriter.flush();
      }

      //直到所有内容都写到HDFS中,则提交事务,表示本次输出全部完毕
      transaction.commit();

      if (txnEventCount < 1) {
        return Status.BACKOFF;
      } else {
        sinkCounter.addToEventDrainSuccessCount(txnEventCount);
        return Status.READY;
      }
    } catch (IOException eIO) {
      transaction.rollback();
      LOG.warn("HDFS IO error", eIO);
      return Status.BACKOFF;
    } catch (Throwable th) {
      //异常回滚事务
      transaction.rollback();
      LOG.error("process failed", th);
      if (th instanceof Error) {
        throw (Error) th;
      } else {
        throw new EventDeliveryException(th);
      }
    } finally {
      transaction.close();
    }
  }
```
