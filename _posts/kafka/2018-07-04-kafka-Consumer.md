---
layout: post
title:  "Kafka Consumer"
date:   2018-07-01 13:31:01 +0800
categories: kafka
tag: [kafka]
---

* content
{:toc}


##  KafkaConsumer    

### 概述  

#### 线程不安全  

KafkaConsumer 是一个线程不安全的接收器器.  
KafkaConsumer 内部定义了acquire方法,用以检测KafkaConsumer实例是否同时运行在多个线程中.(比对当前线程与前一个线程是否一致,如果不一致就抛出ConcurrentModificationException)  

注意 acquire 仅仅检测(抛出异常)而不是互斥锁(阻塞线程等待同步).也就是KafkaConsumer是完全不支持多线程共享的  

所以 KafkaConsumer 常用的模式有两种  
* 在多个线程中使用独立的KafkaConsumer实例  
* 在一个线程使用KafkaConsumer实例,然后将结果多线程处理  

#### 消费者与消费组  

#### 消费订阅    

消费基于Topic进行订阅,  
但在Kafka中Topic只类似一个逻辑概念.无论存储,偏移量还是副本等等最终都是落在分区上的.所以最终消费的是TopicPartition   

**消费实例与分区**  
一个Kafka分区(TopicPartition),最终只会交给一个消费者实例(同消费组下),这里有一个注意点是 `TopicPartition数量` 与 `消费实例数`  
* 如果消费实例等于分区数 这是一种理想情况  
* 如果消费实例数大于分区数  那么某些消费实例将永远也拿不到可供消费的分区  
* 如果消费实例数小于分区数  那么某些消费实例将承担更多消费任务  

**多线程与分区有序**  
因为Kafka只能保持分区有序(全局无序),如果是严格有序场景,需要保证某一个分区数据全部交由某一个线程独立执行  

#### 消费偏移量  

kafka分区消费非常依赖消费偏移量,kafka消费分区的本质是从全量分区上根据偏移量拉取需要的数据  
旧版的kafka偏移量存于ZK`consumers/${group.id }/offsets/${topicName}/$partitionld｝`节点  
新版的kafka会将偏移量作为一个主题 `consumer offsets`进行存储.  

消费偏移量有两种提交方式  自动提交 和 手动提交  
需要注意的是,自动提交丢失和大量重复消费风险,一般会选择手动提交  

`auto.offset.reset`  自动偏移量重置  
这个参数是指消费者没有提交偏移量时,自动将偏移量重置.可选  latest 和 earliest  
* latest  
如果没有提交偏移量,将自动从最新开始消费(即忽略历史数据)  
* earliest  
如果没有提交偏移量,将自动从最老开始消费(即从头消费)  

#### 消费平衡  

消费平衡是指消费者加入或退出消费组,导致重新分配分区给所有消费者的过程.在消费平衡过程中,消费者将暂时无法拉取消息  
消费平衡是一种客户端概念.消费平衡是指如果监听到需要消费平衡,向服务端(ZooKeeper)重新刷取可用分区,在重新刷取到可用分区之前,客户端会暂时阻塞拉取消息  

会产生消费平衡的情况有  
* 新的消费者加入消费组  
* 消费者从消费组退出(无论正常或异常退出,心跳退出等等)  
* 消费者取消对某个Topic的订阅  
* Topic下增加新分区  
* leader失效,新的leader被选举  

#### 重要配置  

##### 消费组  

`group.id`  
消费组.必须设定,偏移量将以消费组为单位进行消费  
`max.poll.records`   
消费限流,一次拉取的最大消费记录数  
`max.poll.interval.ms`   
消费拉取最大间隔时间,超过此时间未拉取数据,服务端会认为消费者已丢失而触发消费平衡  

`request.timeout.ms`  
客户端发送请求后最大等待超时时间  
`heartbeat.interval.ms`  
客户端发送心跳间隔时间  
