---
layout: post
title:  "Spark-Shuffle"
date:   2018-10-11 13:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}


## 概述  

Spark的Shuffle混洗本质上就是同Key分组,是将散落在不同原始分区(Partition)的同Key数据按照Key进行重分组,其原理和思想与Hadoop的Shuffle完全相同  

Spark的Shuffle在Hadoop基础上有了一些演进  
Hadoop的Shuffle主要是Sort-Shuffle思路.即在Map阶段将每个分区数据按照Key排序写入 

## Shuffle的产生  

根据RDD的依赖关系,如果是ShuffleDependecy就会产生ShuffleStage=>ShuffleTask  
RDD转换如果是有下列转换,其依赖关系就会是ShuffleDependecy  
* ByKey系操作  
* join操作(需要区分情况)  
如果join之间分区数和分区函数相同(分区协同),就不会产生Shuffle  
除此之外的join就会产生Shuffle  

## shuffle代价  

shuffle是一种IO和CPU都高耗的行为  
* shuffle往往伴随着大量的磁盘文件读写  
* shuffle会产生大量的节点间数据传输,带来网络IO  
* shuffle往往会伴随着排序,序列化与反序列化,压缩与解压缩过程  


## HashShuffleManager  

最早时,Spark是用Hash-Shuffle来完成Shuffle过程  
HashShuffleManager 在Spark2.X中最终被弃用  

### 核心思路  

* Map阶段先探知Reduce个数  
* 在Map阶段按照Reduce分好组(`Key Hash ReduceCount`),每组独立成为一个文件  
* Reduce直接拉取属于自己的文件  

### 优势  

Hash-Shuffle的最大优势在于没有排序过程,这也是Spark抛弃Hadoop-Sort-Shuffle的初衷  
Spark认为大部分场景下排序是没有意义的(事实的确如此,一般shuffle操作不关心内部是否排序)  
采用Hash-Shuffle的可以完全避开排序过程,节省计算量(大数据量下就是O(n)消耗也是比较大的)  

### 劣势  

Hash-Shuffle虽然节省了排序过程,但带来了另一个问题就是`文件数过多`  
一个Hash-Shuffle使用到的文件数是 `Map数量*Reduce数量`  
这是因为Hash-Shuffle的算法就是在每个Map中按照Reduce的数分组  
假设1000个Map,1000个Reduce 就会产生 100W个文件(意味着100W个文件句柄),这里消耗是非常恐怖的  

### Hash-Shuffle优化  

#### consolidate 机制  

通过`spark.shuffle,consolidateFiles`开启(默认false)  
consolidate机制核心在于引入ShuffleFileGroup  
* 不再为每一个Task创建一个文件,而是创建一个ShuffleFileGroup  
ShuffleFileGroup中会包含多个磁盘文件(等同ReduceTask个数)  
* 一批MapTask执行后通过ShuffleFileGroup写入文件,之后另一批MapTask启动时会继续复用这个ShuffleFileGroup而不会另外新开文件  

## Sort-Shuffle  

因为Hash-Shuffle的恐怖文件句柄问题,Spark最终再一次回到了Hadoop的解决方案    
Sort-Shuffle 现在是Spark的默认shuffle处理方式,内部上`SortShuffleManager`有三种策略  

#### bypass机制  

bypass机制与Hash-Shuffer非常类型,类似Hash与Sort的结合版  
* 首先沿用Hash思路,按Key分组为Bucket不排序,每一个Bucket写出为一个文件  
到这里与Hash-Shuffle完全相同  
* 多个Bucket会合并为一个文件,并生成Index  
bypass与Hash-Shuffle唯一的区别就是最终合并为一个文件  

bypass机制实质是在解决Sort-Shuffle的句柄太多弊端,非常简单的分段思想  
Sort-Shuffle解决其实非常出色,唯一的问题在于Map或Reduce太多,文件句柄膨胀,但在Map或Reduce数量不多,文件句柄膨胀不起来的时候,Hash-shuffle是可以工作的很好的  
所以才有这个特殊分段条件 `spark.shuffle.sort.bypassMergeThreshold`  

bypass的核心思想是  
用更多的可以接受的文件数,来换取排序消耗,这中间就看如何平衡选择了  

#### SerializedShuffleHandle   

序列化Shuffle现在还是不可用方案(Spark内部关闭了,堆外排序还在实验阶段)   
序列化Shuffle作用于二进制数据而不是java对象本身  
使用序列化的好处在于  

* 减少了内存消耗和GC开销  
* 使用专门的高效排序器(ShuffleExternalSorter)进行排序  
它内部的排序数组对每条数据只使用8字节,可以容纳更多的数据  
* 在之后的合并溢写,压缩甚至传输时都可以直接使用序列化结果  

#### BaseShuffleHandle

BaseShuffleHandle是shuffle常规意义上的解决方案 

整个Map端的Shuffle过程就是一个LSM算法  
* Map阶段分块分段(受内存限制)读取,溢写文件到磁盘(溢写文件内保持Key有序)  
* 多个溢写文件归并排序为一个最终文件,依然保持有序  
同时为这一个文件生成Index文件,标记这个文件的Key分布(每个Key的起始和结束Offset)  
* 每个Reduce读取时通过Index从最终文件中拉取数据  

## Shuffle核心源码  

Shuffle的核心在 `SortShuffleManager`

```scala
//Shuffle策略选择  
override def registerShuffle[K, V, C](
      shuffleId: Int,
      numMaps: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
      // Bypass使用条件  
      // 1.分区少于 sparle.shuff.sort.bypassMergeThreshold (默认200)
      // 2.无聚合场景(没有使用聚合算子)
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // 如果Bypass未能起效,接下来尝试 序列化-Shuffle  
      // 序列化-Shuffle的使用条件
      // 1. 序列化方式必须支持 relocation 方式 
      //       也就是可以在无需反序列化的前提下完成排序  
      // 2. 无聚合场景
      // 3. 分区数量在16777215(2的23次)以内
      // 4. dependency.serializer.supportsRelocationOfSerializedObjects  
      //    事实上这个条件固定为false,序列化Shuffle所用的堆外排序还在实验中
      new SerializedShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // 如果 序列化-Shuffle 也未能起效
      // 最后选择使用 Base-Shuffle
      new BaseShuffleHandle(shuffleId, numMaps, dependency)
    }
  }

// 根据shuffle策略构造Shuffle-Writer
override def getWriter[K, V](
      handle: ShuffleHandle,
      mapId: Int,
      context: TaskContext): ShuffleWriter[K, V] = {
    numMapsForShuffle.putIfAbsent(
      handle.shuffleId, handle.asInstanceOf[BaseShuffleHandle[_, _, _]].numMaps)
    val env = SparkEnv.get
    handle match {
      case unsafeShuffleHandle: SerializedShuffleHandle[K @unchecked, V @unchecked] =>
        new UnsafeShuffleWriter(
          env.blockManager,
          shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
          context.taskMemoryManager(),
          unsafeShuffleHandle,
          mapId,
          context,
          env.conf)
      case bypassMergeSortHandle: BypassMergeSortShuffleHandle[K @unchecked, V @unchecked] =>
        new BypassMergeSortShuffleWriter(
          env.blockManager,
          shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
          bypassMergeSortHandle,
          mapId,
          context,
          env.conf)
      case other: BaseShuffleHandle[K @unchecked, V @unchecked, _] =>
        new SortShuffleWriter(shuffleBlockResolver, other, mapId, context)
    }
  }  
  
  // 构造Shuffle-Reader  
  // 这个没有区别,因为每种shuffle策略最终都会转化为ShuffleBlock  
  override def getReader[K, C](
      handle: ShuffleHandle,
      startPartition: Int,
      endPartition: Int,
      context: TaskContext): ShuffleReader[K, C] = {
    new BlockStoreShuffleReader(
      handle.asInstanceOf[BaseShuffleHandle[K, _, C]], startPartition, endPartition, context)
  }
```

## Shuffle倾斜  

Shuffle的本质是按Key分组,所以是数据倾斜的重点优化点,事实上基本所有的倾斜问题都是Shuffle导致的  

### 倾斜点  

数据倾斜时,首先是定位倾斜点,找出  
* 是同Key倾斜还是同区倾斜  
* 如果是同Key倾斜,定位出是哪一个Key产生了倾斜  

定位倾斜点是处理倾斜的重要指向,定位上可以遵循  
* 根据SparkUI的执行过程定位倾斜Task  
* 根据Task与DAG,定位倾斜源码点  
* 根据倾斜源码点,定位是什么数据的什么操作产生了倾斜  
* 根据数据和操作,对数据进行采样或者Key统计  

### 同区倾斜  

同区倾斜是指倾斜Key本身不同,但因为哈希冲突导致大量数据涌入了同一分区  
解决办法也非常简单  
因为是哈希冲突导致的,`调整分区数`就可以了  

### 同Key倾斜  

同Key倾斜指倾斜点本身是同一个Key,这个时候调整分区数据是没有意义的,因为同Key必然同分区  

常用的解决思路有  
* 过滤倾斜键  
这种要求业务支持,比较适用于null或异常键倾斜情况  
* 去shuffle  
比如Boardcast-Join,这需要大小表等数据现状上有支持  
* 随机打散  
这是解决倾斜最经典的思路,打散后聚合类需要做二次聚合,Join系需要做小表膨胀  

## Shuffle优化  

shuffle过程是Spark内置过程,在应用层面很难干预,所以Shuffle优化一般是配置优化,让Spark的内置Shuffle过程运行的更好  

### Map 

* `spark.shuffle.file.buffer` 文件溢写缓冲大小 默认32K   
如果内存充足可以适当调大,这可以减少Map端文件溢写次数  

### Reduce

* `spark.shuffle.io.maxReties` 默认3次  
这是拉取请求的错误重试次数,一般可以适当调大(30次)  
因为网络波动或目标executor处于GC过程导致不可访问,shuffle请求很可能不成功,调大可以提高稳定性  
*  `spark.shuffle.io.retryWait` 默认5S  
这是拉取请求的最大超时时间,一般可以适当调大(30S),原因同上  
* `spark.reducer.maxSizeInFlight` 默认48M  
拉取请求的缓冲块(一次拉取多少大小的数据),如果内存充足可以适当调大,可以减少拉取次数  
* `spark.reducer.maxBlocksInFlightPerAddress` 默认`Int.Max`  
每个主机(Address)可以被多少个Reducer主机(Address)拉取数据,在大集群中可以适当调低来降低某个热点executor的压力  
* `spark.reducer.maxReqsInFlight`  默认 `Int.Max`  
每个主机(Address)可以接收多少个Reducer拉取请求,大集群中可以适当调低来降低某个热点executor的压力  

### 内存&压缩  

* Spark1.6 `spark.shuffle.io.memoryFraction` 默认0.2 适当调大   
Spark2.X  `spark.memory,storageFracation` 默认0.5,适当压低  
Spark2.X是动态内存管理,Shuffle是占据执行内存的一部分,通过压低存储内存可以调高执行内存  
* `spark.shuffle.compress` Map输出压缩,默认true  
* `spark.shuffle.spill.compress` 溢写压缩,默认true  
* `spark.io.compression.codec` 默认lz4,可以考虑使用`snappy`   
Shuffle默认使用压缩,压缩方式使用的`spark.io.compression.codec`  
* `spark.shuffle.consolidateFiles` 默认false ,可以设为true  
这是在Spark2.x已经废弃,但老版本可以使用的优化,在使用优化版的Hash-Shuffle  


