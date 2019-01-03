---
layout: post
title:  "kafka Producer"
date:   2018-07-01 13:31:01 +0800
categories: kafka
tag: [kafka]
---

* content
{:toc}


##  KafkaProducer    

### 概述  

KafkaProducer 是一个线程安全的发送器.并且是建议在多线程环境下保持KafkaProducer的单例共享.(为了共用缓冲池,下文详述)  

### 实现原理  

KafkaProducer(新版) 的消息发送是一个异步模式  
为了维持这个异步模式,Kafka中包含双端队列,缓冲池,TCP连接对象和KafkaThread的守护线程  
这就是建议在多线程环境下使用KafkaProducer单例共享的原因,在多线程中共享这些组件.而不是在每个线程中再创建诸如TCP连接对象,守护线程等等  

* Serializer对K,V进行序列化,最终成为一个Record  
* Partitioner为Record选择合适的分区(TopicPartition)  
* Record聚集成RecordBatch,加入目标TopicPartition所在的双端队列(每个TopicPartition一个双端队列,每个双端队列的实际元素是RecordBatch)  
* Sender(包含TCP连接的守护线程),从双端队列头取出Batch,发送给服务器  
* 发送完毕后调用Batch的回调函数.(Batch的回调等价于每一个Record的回调)  

####  双端队列缓冲  

KafkaProducer中消息的发送不是即时发送,会首先交给一个双端队列中

*** RecordBatch ***  
队列中的元素为RecordBatch.也就是Record的合并结果.由batch.size或linger.ms共同完成   

*** 队列保持与kafka分区一一对应 ***  
即每一个TopicPartition都会有一个双端队列  
也就是加入队列前会检查Meta信息.因为事实上Producer的双端队列是保持与kafka分区一一对应关系,这个对应关系是依赖producer客户端对Meta的本地缓存(各分区leader与副本,数量位置等等).所以会有一个发送之前会有一个waitOnMetadata,检查当前Meta是否可用,如果不可用会阻塞发送直到Meta信息的刷新  

*** 双端 ***    
* 消息的正常发送从队尾入队  
* 消息的重试由队头入队  

#### Sender    

Kafka底层连接是一个与服务器维持的TCP连接,并会构造一个守护线程在后台不断轮询将消息发送给服务器   


## 核心源码  

```scala
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        // first make sure the metadata for the topic is available
        long waitedOnMetadataMs = waitOnMetadata(record.topic(), this.maxBlockTimeMs);
        long remainingWaitMs = Math.max(0, this.maxBlockTimeMs - waitedOnMetadataMs);
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer");
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer");
        }
        int partition = partition(record, serializedKey, serializedValue, metadata.fetch());
        int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
        ensureValidRecordSize(serializedSize);
        tp = new TopicPartition(record.topic(), partition);
        long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
        log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = this.interceptors == null ? callback : new InterceptorCallback<>(callback, this.interceptors, tp);
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    }
```

##  重要配置     

###   消息确认机制

`acks`  
* 0 表示无确认,速度最快  
* 1 表示确认至leader节点  
确认至leader有一定的保证性,但在leader突然崩溃时依然会有丢失风险,这里务必注意  
* all 表示确认至所有副本节点  

### retry  

`message.send.max.retries`  默认3  
消息因为各种原因发送失败的重试次数  

`retry.backoff.ms`  默认100ms  
重试间隔.注意生产者的每次重试都会刷新一次MetaData(因为错误原因可能是leader丢失的重新选举),需要一定的时间  

### 批量发送  

使用批处理将会造成消息发送延时,但会极大的提高发送吞吐量  

`batch.size` 批处理字节数 默认16384(16K)  
积累消息直到总字节数达到这个阈值时才会发送一次,  
batch.size不是越大越好,过大的batch.size会造成生产者端的内存浪费  

`linger.ms` 默认0 
积累消息直到时间达到这个阈值才会发送一次  

总字节数和消息时间这两个条件不冲突,Kafka在这两个条件任意满足一个即发送批次  


`buffer.memory`  默认33554432(32M)  
生产者的内存缓冲区,用它来暂时缓冲需要发送到服务器的消息.必须保证消息产生的速度小于消息缓冲到发送的速度,否则要么会造成阻塞要么会抛出异常. 
buffer.memory过小非常影响性能,会频繁block.过大则性能没什么提升造成浪费,建议可以与`batch.size`匹配倍数  


`receive.buffer bytes` 和 `send buffer.bytes`  
TCP 接收和发送缓冲大小. 一般情况不用考虑,但如果是跨数据中心网络,可以适当调高一些(两者一致)


### 超时  

`request.timeout.ms` 30000   
消息发送等待服务器响应的最大超时时间   

`metadata.fetch.timeout.ms`  60000  
获取元数据(例如leader)等待服务器响应的最大超时时间  

`timeout.ms` 30000   
与ACK机制相关,等待副本返回响应的超时时间  

`max.block.ms` 60000   
生产者发送消息的本地阻塞超时时间.这个阻塞可能是缓冲池满还未来得及发送或者正等待元数据同步等等  

### 压缩  

`compressed.codec`  默认none  
可以有gzip 和 snappy  
`compressed.topics` 针对topic的压缩  

### 有序化  

`max.in.flight.requests.per.connection`  
指定在发送后收到响应之前还可以发送多少个消息  
这个参数会最终影响有序性,因为a,b顺序可能因为a的重试发送而最终变成b,a,设定这个参数为1可以强一致的保证发送顺讯,但这会导致发送的吞吐性能降低非常多,所以一定谨慎使用  


