---
layout: post
title:  "flume-源码解析-Sink"
date:   2018-07-10 13:31:01 +0800
categories: flume
tag: [flume,源码解析]
---

* content
{:toc}


# 核心流程  

Sink核心流程是 SinkRunner 不断调用 SinkProcessor的Process方法  
* 根据配置的 SinkProcessor,会使用不同的策略来选择 Sink.  
SinkProcessor 有3种,默认DefaultSinkProcessor
* 调用选择 Sink 的 Process 方法  

# 核心方法-Process  

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

        // reconstruct the path name by substituting place holders
        String realPath = BucketPath.escapeString(filePath, event.getHeaders(),
            timeZone, needRounding, roundUnit, roundValue, useLocalTime);
        String realName = BucketPath.escapeString(fileName, event.getHeaders(),
          timeZone, needRounding, roundUnit, roundValue, useLocalTime);

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
