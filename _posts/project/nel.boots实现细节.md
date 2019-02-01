
* content
{:toc}

## RabbitMQ  

RabbitMQ的特点是单条消息.(单条消费,单条ACK),所以采用N+M模式  
* 使用多线程更好的消费信息,所以必然是M个线程级别消息消费  
* 物理连接会耗用服务器资源,所以必然不可能是每个消费线程占用一个物理连接(依据一可知,消费线程数量可以是非常恐怖的)  

主要基于`ArrayBlockingQueue`实现  
根据`处理线程数`,作为`ArrayBlockingQueue`的队列缓冲大小  
然后根据`连接线程数`和`处理线程数`,分别构建指定数量的线程依托`ArrayBlockingQueue`写入消息和消费消息  
`ArrayBlockingQueue`内置封装了如果队列缓冲溢出则阻塞生产线程,如果队列缓冲为空则阻塞消费线程  

`ArrayBlockingQueue`的实现原理中核心是`juc.ReentrantLock` 和 `Array`  
* `ArrayBlockingQueue`中的缓存实现是一个线程安全的数组(`ReetrantLock.lock`)  
线程的数组实现利用的双游标(`入队游标`,`出队游标`)  
* `ReetrantLock`分别为`入队`和`出队`构建两个`Condition`  
`入队`检测队列满则`enqueue.condition.await`,同时`dequeue.condition.signal`唤醒一个出队线程  
`出队`检测队列空则`dequeue.condition.await`,同时`enqueue.condition.signal`唤醒一个入队线程  

## KafkaMQ  

KafkaMQ的特点是批量消息(TopicPartition,TopicPartition-Offset提交),所以采用1+M模式  
这里是完全自封装的(借鉴了Spark的Kafka分区感知机制)  
* Kafka的消费实例有`Kafka-消费平衡`机制,所以必然是单一实例  
* 使用多线程更好的消费信息,所以必然是M个线程级别消息消费,但此时的M数量是有要求的.为了保证批量数据的有序,必须精确控制每一个线程完整消费一个`TopicPartition`(因为保证`TopicPartition`内有序,但如果被跨线程分割就不能完全保证了)  
所以会引入Kafka的`分区感知`机制,即根据`连接线程`单次`poll`结果的`topicPartition.partitions.size`数量来动态变更线程池(`setCorePoolSize`,`setMaximumPoolSize`)  
使用线程池是因为既可以其方便控制处理线程的上限,也可以利用其缓慢释放(线程空闲5分钟被杀死)的特点在数据繁忙时避免大量的线程重建  
* 每一个poll必须全部处理完成,在这之前连接线程会阻塞.  
连接线程自旋阻塞,并监控所有`TopicPartition`的线程执行结果是否完毕  
* 处理完成后,自动的OffSet管理  
以TopicPartition为单位提交Offset.  `Map( callback.topicPartition -> new org.apache.kafka.clients.consumer.OffsetAndMetadata(callback.lastOffset + 1))` 构建 目标topicPartition + (目标最终lastOffset+1)
通过 `commitAsync` 提交  

## 自定义JDBC数据源  

Spark数据源的核心  
* Relation BaseRelation  
* JDBC-RDD的分区实现  
* SQL再包装分页语句  

## HBase  

### CRUD 

### Spark  
利用`org.apache.hadoop.hbase.mapreduce.TableInputFormat` 和 `org.apache.hadoop.hbase.mapreduce.TableOutputFormat` 来读写HBase

读  
支持列裁剪,行过滤只支持RowKey范围  

```scala
val hbaseConf = HBaseConfiguration.create()
hbaseConf.set(TableInputFormat.INPUT_TABLE, options.table)
hbaseConf.set(TableInputFormat.SCAN_COLUMNS, queryColumns);
hbaseConf.set(TableInputFormat.SCAN_ROW_START, startRowKey);
hbaseConf.set(TableInputFormat.SCAN_ROW_STOP, endRowKey);

val hbaseRdd = sqlContext.sparkContext.newAPIHadoopRDD(
  hbaseConf,
  classOf[org.apache.hadoop.hbase.mapreduce.TableInputFormat],
  classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],
  classOf[org.apache.hadoop.hbase.client.Result]
)
val reolver = new HBaseRowResolver(hbaseTableSchema)
hbaseRdd.map(tuple => tuple._2).map(reolver.resultToRow(_))
```
写

写的时候必须注意三排序  即首先多条数据之间按照RowKey升序,每条数据内部按照列族升序序,列族内再按照列升序的方式完成排序后进行写入  

```scala
val hbaseConf = HBaseConfiguration.create()

hbaseConf.setInt("hbase.mapreduce.bulkload.max.hfiles.perRegion.perFamily", 1024);
hbaseConf.set(TableOutputFormat.OUTPUT_TABLE, options.table)


val sortSchemas = data.schema.filter(x => x != keyColumn).sortBy(x => x.name).map(x=> {
  val c = HBaseSchemaResolver.resolveColumn(x.name)
  HBaseSchema(c._1,Bytes.toBytes(c._1),c._2,Bytes.toBytes(c._2),x.dataType)
})

val job = Job.getInstance()
job.setMapOutputKeyClass(classOf[ImmutableBytesWritable])
job.setMapOutputValueClass(classOf[KeyValue])
job.setOutputFormatClass(classOf[HFileOutputFormat2])

usingPattern(ConnectionFactory.createConnection(hbaseConf))(conn => {
  val tableName = TableName.valueOf(options.table)
  val regionLocator = conn.getRegionLocator(tableName)
  val table = conn.getTable(tableName)
  HFileOutputFormat2.configureIncrementalLoad(job, table, regionLocator)
})
```

## Spring的各种原理  

## 线程池原理和一个轻量级线程池

## 缓存

### 缓存穿透  



### 缓存雪崩  

缓存雪崩是指在某个时间点或某个相对集中的时间段内,缓存因为大量过期导致请求大量的穿透缓存进入数据层,导致雪崩效应  



### 一致性  

## 高并发架构    

数据库问题,写入,读写分离  
缓存
负载均衡,异步处理与并行处理

### 数据库  

#### 高并发写入  

订单号采用的自增Id,一秒订单超过2000  
这样的好处是始终从尾页追加避免页分裂,劣势是尾页有数据热点问题  
解决的办法是多表写入然后再批量移入  
* 外部订单号取模写入10张暂存表中的一张表  
* 每个暂存表有一个独立服务批量读取整体移入订单主表  

#### 读写分离  

主库  
* 订单创建时间维度分区表  
* 订单表除了主键聚集索引之外,不使用任何索引,主库只能使用根据订单号查询单条订单  

从库  
* 所有查询以及辅助索引全部由从库解决  

### 缓存  

#### Redis  

Redis缓存主要用来做订单缓存  
* 雪崩   
基于订单粒度的Redis缓存不太可能出现雪崩  
* 穿透  
主要针对的是订单号不存在的情况,这时会写一个特殊字串`*NIL*`作为标记并进行正常缓存.同时设置这种空存在标志型的缓存过期时间设置为30秒(正常的订单缓存为5分钟)  
* 一致性  
双删机制,但总的来说对于一致性要求不高  
对于不存在订单30秒后必查一次(对外要求5分钟后订单不存在才视为不存在)  
对于存在的订单只有初始化,充值中和终结状态,只要最终一致(最终得到终结状态),中途出现几次脏的充值中没什么影响  
对于一致性不高的情况甚至可以使用多级缓存(内存缓存15秒),不过从现在来看足够用了  

####  SqlServer依赖缓存  

SqlServer依赖缓存用来做配置级别的缓存(比如通道配置等等)  
SqlServer依赖缓存是一种SqlServer特有机制,它是在客户端就维持一个与SqlServer的长连接,客户端会在内存中维持一个DataTable来映射一个某一张表,它的机制是SqlServer会把这张表的新增修改删除全部推送到客户端中  



### 服务器优化的常规思路  

* 空间换时间  
对热点数据缓存，减少数据查询时间  
* 分而治之   
将大任务切片，分开执行。HDFS、MapReduce就是这个原理  
* 异步处理  
若业务链中有某一环节耗时严重，则该环节将拉长业务链的整体耗时。可以将耗时业务采用消息队列异步化，从而缩短业务链耗时  
* 并行处理   
采用多进程、多线程同时处理，提升处理速度  
* 离用户更近一点  
如CDN技术，将静态资源放到离用户更近的地方，从而缩短请求静态资源的时间  
* 提升可扩展性  
采用业务模块化、服务化的手段，提升系统的可扩展性，从而可根据业务需求实现弹性计算

CPU 
CPU使用率过高的原因：

计算量大
非空闲等待
过多的系统调用
过多的中断
内存 
内存使用率过高的原因：

过多的页交换
可能存在内存泄露
IO 
IO繁忙的原因：

读写频繁 
磁盘的读写过程是物理动作，频繁的读写势必会使IO来不及处理。
https://zhuanlan.zhihu.com/p/34266039


Flume 架构 双层意义  