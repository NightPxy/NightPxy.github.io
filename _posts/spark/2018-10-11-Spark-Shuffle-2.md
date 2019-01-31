---
layout: post
title:  "Spark-Shuffle-2"
date:   2018-10-11 14:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}


## 概述  

Sorted-Based-Shuffle 的核心是是借助`ExternalSorter`把每个`ShuffleMapTask`的结果输出,排序输出到一个文件`FileSegmentGroup`中,为了方便后续Reduce的读取,还会产生一个`index`文件用来标记Key的范围  
这里再来仔细研究下这个过程  


### SortShuffleWriter

```scala
private[spark] class SortShuffleWriter[K, V, C](
    shuffleBlockResolver: IndexShuffleBlockResolver,
    handle: BaseShuffleHandle[K, V, C],
    mapId: Int,
    context: TaskContext)
  extends ShuffleWriter[K, V] with Logging {
   
   //SortShuffleWriter的核心实现是借助的ExternalSorter
   private var sorter: ExternalSorter[K, V, _] = null
      
   override def write(records: Iterator[Product2[K, V]]): Unit = {
    //根据是否需要Map端聚合构建不同的外部排序器
    //本质上都是ExternalSorter
    //如果需要Map端聚合,则传入聚合计算函数,同时设定为Key排序
    //如果不需要Map端聚合,则设定聚合计算为None,同时设定为不排序
    sorter = if (dep.mapSideCombine) {
      require(dep.aggregator.isDefined, "Map-side combine without Aggregator specified!")
      new ExternalSorter[K, V, C](
        context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
    } else {
      // 非Map端聚合情况下,不传入聚合也不关心排序
      // 就算有SortByKey算子之类要求排序场景也将在Reduce端完成
      new ExternalSorter[K, V, V](
        context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
    }
    //外部排序器的写入方法,这是核心中的核心,后面详述
    sorter.insertAll(records)

    // 构建Map端输出文件.
    // 注意只有一个文件被构建,这也是Sort-Shuffle的核心所在,减少Map端的文件输出
    val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
    val tmp = Utils.tempFileWith(output)
    try {
      //构建Map-Shuffle结果
      //Map-Shuffle的结果视为存储体系中的一个Block(ShuffleBlock)  
      //这是与ResultTask区别所在  
      //  ResultTask完成后上报的是结果
      //  ShuffleTask完成后最终上报的不是结果本身,而是Block-MetaData  
      val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
      val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
      shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
      mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
      }
    }
  }
  }
```

### ExternalSorter  

```scala
private[spark] class ExternalSorter[K, V, C](
    context: TaskContext,
    aggregator: Option[Aggregator[K, V, C]] = None,
    partitioner: Option[Partitioner] = None,
    ordering: Option[Ordering[K]] = None,
    serializer: Serializer = SparkEnv.get.serializer)
  extends Spillable[WritablePartitionedPairCollection[K, C]](context.taskMemoryManager())
  with Logging {
  
  //ExternalSorter中核心数据结构
  //map 处理Map端聚合情况
  //buffer 处理非Map聚合情况
  @volatile private var map = new PartitionedAppendOnlyMap[K, C]
  @volatile private var buffer = new PartitionedPairBuffer[K, C]
  
  //溢写缓冲区大小,默认每32KB溢写一次
  private val fileBufferSize = conf.getSizeAsKb("spark.shuffle.file.buffer", "32k").toInt * 1024
}
```


