---
layout: post
title:  "SparkStreaming-DirectKafkaInputDStream-源码解析"
date:   2018-09-01 13:31:01 +0800
categories: spark
tag: [spark,源码解析]
---

* content
{:toc}


## DirectKafkaInputDStream  

是Spark-Streaming里一个非常重要的DStream实现类.  
之所以重要是它是实现与kafka对接的DStream.而在Streaming实际生产中,绝大部分都是kafka  

这里以kafka10中的DirectKafkaInputDStream为例,这个类在spark源码-external-kafka-0-10中  
这里来仔细看看这个DStream的源码实现  

## 创建 DirectKafkaInputDStream

先从DirectKafkaInputDStream的创建入口开始  

```scala
val directKafkaStream = KafkaUtils.createDirectStream[String, String](
  ssc,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)
------------------------------------------------------
// createDirectStream 其实是一个工厂方法,创建一个DirectKafkaInputDStream实例
def createDirectStream[K, V](
      ssc: StreamingContext,
      locationStrategy: LocationStrategy,
      consumerStrategy: ConsumerStrategy[K, V]
    ): InputDStream[ConsumerRecord[K, V]] = {
    val ppc = new DefaultPerPartitionConfig(ssc.sparkContext.getConf)
    createDirectStream[K, V](ssc, locationStrategy, consumerStrategy, ppc)
  }
```
* ssc  
这里ssc其实StreamingContext上下文,无论怎样,DirectKafkaInputDStream都是一个Streaming中的DSream,所以必然会有一个StreamingContext上下文    
* PreferConsistent 
PreferConsistent 是Streaming在对接kafka的本地化策略,下面详述   
* Subscribe[String, String](topics, kafkaParams)   
从字面意义,这是订阅设置.订阅什么呢?其实包装如何访问kafka系统的描述.kafka-topic   
kafkaParams里是kafka地址等等相关  

## DirectKafkaInputDStream

```scala
private[spark] class DirectKafkaInputDStream[K, V](
    _ssc: StreamingContext,
    locationStrategy: LocationStrategy,
    consumerStrategy: ConsumerStrategy[K, V],
    ppc: PerPartitionConfig
  ) extends InputDStream[ConsumerRecord[K, V]](_ssc) with Logging with CanCommitOffsets {
  ......
}
```

### LocationStrategy 

出于性能的考虑,spark会在executor上缓存kafka的消费者实例(类似长连接),而不是在每个批次上都重新再去创建.
在kafka中,消费者实例实际是与分区绑定的,这个本地化策略就是描述这些消费者实例(对应kafka分区)怎样在应用的多个executor上进行分布   
spark提供了三种分布策略  

```scala

//倾向于在该节点上安排KafkaLeader对应的分区
//这是一种专门用在executor与kafka-broker在同一节点的情况
//实际情况基本不太可能,所以这种用的非常少
case object PreferBrokers extends LocationStrategy

//这是用的最多的本地化策略
//这代表所有消费者实例将在每个executor上尽可能均匀的分布
case object PreferConsistent extends LocationStrategy

//这是专门用来处理kafka的分区倾斜的情况
//即某个kafka分区数据特别庞大,比较有必要对这个分区消费者实例独占executor
//所以这种本地化策略,需要我们指定kafka分区与executor主机的映射关系
case class PreferFixed(hostMap: ju.Map[TopicPartition, String]) extends LocationStrategy
```

### ConsumerStrategy[K, V] 

```scala
abstract class ConsumerStrategy[K, V] {
  /**
   * Kafka <a href="http://kafka.apache.org/documentation.html#newconsumerconfigs">
   * configuration parameters</a> to be used on executors. Requires "bootstrap.servers" to be set
   * with Kafka broker(s) specified in host1:port1,host2:port2 form.
   */
  def executorKafkaParams: ju.Map[String, Object]

  /**
   * Must return a fully configured Kafka Consumer, including subscribed or assigned topics.
   * See <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Kafka docs</a>.
   * This consumer will be used on the driver to query for offsets only, not messages.
   * The consumer must be returned in a state that it is safe to call poll(0) on.
   * @param currentOffsets A map from TopicPartition to offset, indicating how far the driver
   * has successfully read.  Will be empty on initial start, possibly non-empty on restart from
   * checkpoint.
   */
  def onStart(currentOffsets: ju.Map[TopicPartition, jl.Long]): Consumer[K, V]
}
```

**SubscribePattern**

```scala
private case class SubscribePattern[K, V](
    pattern: ju.regex.Pattern,
    kafkaParams: ju.Map[String, Object],
    offsets: ju.Map[TopicPartition, jl.Long]
  ) extends ConsumerStrategy[K, V] with Logging {

  def executorKafkaParams: ju.Map[String, Object] = kafkaParams

  def onStart(currentOffsets: ju.Map[TopicPartition, jl.Long]): Consumer[K, V] = {
    val consumer = new KafkaConsumer[K, V](kafkaParams)
    consumer.subscribe(pattern, new NoOpConsumerRebalanceListener())
    val toSeek = if (currentOffsets.isEmpty) {
      offsets
    } else {
      currentOffsets
    }
    if (!toSeek.isEmpty) {
      // work around KAFKA-3370 when reset is none, see explanation in Subscribe above
      val aor = kafkaParams.get(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG)
      val shouldSuppress =
        aor != null && aor.asInstanceOf[String].toUpperCase(Locale.ROOT) == "NONE"
      try {
        consumer.poll(0)
      } catch {
        case x: NoOffsetForPartitionException if shouldSuppress =>
          logWarning("Catching NoOffsetForPartitionException since " +
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG + " is none.  See KAFKA-3370")
      }
      toSeek.asScala.foreach { case (topicPartition, offset) =>
          consumer.seek(topicPartition, offset)
      }
      // we've called poll, we must pause or next poll may consume messages and set position
      consumer.pause(consumer.assignment())
    }

    consumer
  }
}
```

**Subscribe**

```scala
private case class Subscribe[K, V](
    topics: ju.Collection[jl.String],
    kafkaParams: ju.Map[String, Object],
    offsets: ju.Map[TopicPartition, jl.Long]
  ) extends ConsumerStrategy[K, V] with Logging {

  def executorKafkaParams: ju.Map[String, Object] = kafkaParams

  def onStart(currentOffsets: ju.Map[TopicPartition, jl.Long]): Consumer[K, V] = {
    val consumer = new KafkaConsumer[K, V](kafkaParams)
    consumer.subscribe(topics)
    val toSeek = if (currentOffsets.isEmpty) {
      offsets
    } else {
      currentOffsets
    }
    if (!toSeek.isEmpty) {
      // work around KAFKA-3370 when reset is none
      // poll will throw if no position, i.e. auto offset reset none and no explicit position
      // but cant seek to a position before poll, because poll is what gets subscription partitions
      // So, poll, suppress the first exception, then seek
      val aor = kafkaParams.get(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG)
      val shouldSuppress =
        aor != null && aor.asInstanceOf[String].toUpperCase(Locale.ROOT) == "NONE"
      try {
        consumer.poll(0)
      } catch {
        case x: NoOffsetForPartitionException if shouldSuppress =>
          logWarning("Catching NoOffsetForPartitionException since " +
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG + " is none.  See KAFKA-3370")
      }
      toSeek.asScala.foreach { case (topicPartition, offset) =>
          consumer.seek(topicPartition, offset)
      }
      // we've called poll, we must pause or next poll may consume messages and set position
      consumer.pause(consumer.assignment())
    }

    consumer
  }
}
```

**Assign**

```scala
private case class Assign[K, V](
    topicPartitions: ju.Collection[TopicPartition],
    kafkaParams: ju.Map[String, Object],
    offsets: ju.Map[TopicPartition, jl.Long]
  ) extends ConsumerStrategy[K, V] {

  def executorKafkaParams: ju.Map[String, Object] = kafkaParams

  def onStart(currentOffsets: ju.Map[TopicPartition, jl.Long]): Consumer[K, V] = {
    val consumer = new KafkaConsumer[K, V](kafkaParams)
    consumer.assign(topicPartitions)
    val toSeek = if (currentOffsets.isEmpty) {
      offsets
    } else {
      currentOffsets
    }
    if (!toSeek.isEmpty) {
      // this doesn't need a KAFKA-3370 workaround, because partitions are known, no poll needed
      toSeek.asScala.foreach { case (topicPartition, offset) =>
          consumer.seek(topicPartition, offset)
      }
    }

    consumer
  }
}
```

### PerPartitionConfig  

```scala
abstract class PerPartitionConfig extends Serializable {
  def maxRatePerPartition(topicPartition: TopicPartition): Long
}
```

在createDirectStream这个工厂方法中,创建了一个默认的DefaultPerPartitionConfig.  
这是一个非常重要的东西,那就是kafka的读取限速  
试想一个情况是,如果因为某个原因streaming应用在很长时间没有消费消息,kafka必然会积压大量的消息等待处理.这个时候streaming应用启动后如果重新抓取全部未读取,streaming应用很可能会就此崩掉而再也起不来了.  
这里就需要抓取限速了,这个限速很简单,就是规定一个单次最多抓取多少条记录  

```scala
def createDirectStream[K, V](
      ...
    ): InputDStream[ConsumerRecord[K, V]] = {
    val ppc = new DefaultPerPartitionConfig(ssc.sparkContext.getConf)
    ...
  }
  ----------------------------------------------------------------
  private class DefaultPerPartitionConfig(conf: SparkConf)
    extends PerPartitionConfig {
  val maxRate = conf.getLong("spark.streaming.kafka.maxRatePerPartition", 0)

  def maxRatePerPartition(topicPartition: TopicPartition): Long = maxRate
}
```

可以看见spark已经默认设计了一个限速策略,就是读取配置中设置的最大抓取条数  
`spark.streaming.kafka.maxRatePerPartition` 如果为0,表示不限速

### CanCommitOffsets




```scala
@transient private var kc: Consumer[K, V] = null
  def consumer(): Consumer[K, V] = this.synchronized {
    if (null == kc) {
      kc = consumerStrategy.onStart(currentOffsets.mapValues(l => new java.lang.Long(l)).asJava)
    }
    kc
  }
```
根据前面指定的消费者实例策略













https://blog.csdn.net/qq_36421826/article/details/81660915
https://blog.csdn.net/V_Gbird/article/details/80457064




