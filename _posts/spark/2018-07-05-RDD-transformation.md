---
layout: post
title:  "RDD的transformation"
date:   2018-07-05 13:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}



# transformation 操作  

## 概述  

RDD包含两大类操作  

* **transformation(转换)**   从一个RDD转换得到一个新的RDD  
* **action(动作)**  执行对RDD的计算并返回结果  

**action(动作) => Job**  
> 只有触发一个动作(action),才会真正产生一个执行作业来执行逻辑  

**transformation(转换) Lazy**  
> 所有RDD的转换操作都是Lazy的.也就是转换操作本身只是记录如何转换的过程.  
> 转换操作,必须在某一个动作中被需要,才会真正被执行  


## 转换的依赖关系

转换是从父RDD经过一些转换操作生成一个新的RDD.而RDD本身是封装有对父RDD的依赖关系的  
Spark依据RDD转换中,父RDD和子RDD分区的依赖关系,将转换的依赖分为两类 **宽依赖** 和 **窄依赖**  

### 窄依赖  

窄依赖是指转换后,父RDD的某个或者某些分区只会被子RDD的一个分区使用.(*一对一或者多对一的关系*)  

窄依赖一般出现在map,filter等子分区沿用父分区,不会发生重分区的情况

### 宽依赖

宽依赖是指转换后,父RDD的某一个分区,会被多个子RDD的分区共同使用.(*一对多或多对多*)  

宽依赖的本质是shuffle,所以一般出现在groupByKey,reduceByKey等会发生shuffle重分区的情况  

### 宽依赖与窄依赖的对比  

一般来说,窄依赖会更加有利,这主要体现在:  

* 窄依赖允许集群节点以流水线的形式直接计算所有分区  
宽依赖则必须以shuffle的形式重新分区,在重新分区全部接收后再进行计算  

* 在某个分区发生错误需要重新计算的时候.实际是对依赖的分区进行计算(*宽窄皆同*),但是:  
窄依赖,计算的依赖的分区,利用率是100%的  
宽依赖,因为依赖分区只有一部分的数据是属于它的,但也必须重新计算整个分区.(*属于其它分区的数据白计算的*)  

## spark shuffle 过程  

### 简述  
shuffle过程就是不同节点不同分区的数据进行重新分组分区的过程(详见Hadoop-MapReduce篇)   
shuffle过程本身是一个代价很高的过程,因为它涉及不同节点不同executor的数据复制  

Spark的某些操作可能会引起shuffle过程  
以reducevByKey为例,需要将原始数据中相同的Key分为同一组,而对某一组的数据而言,其可能来自不同的节点不同的分区  

### spark shuffle 与 hadoop shuffle  
spark shuffle 与 hadoop shuffle 是非常类似的,只是在某些细节上稍有不同  

hadoop的shuffle结果是分区有序的,且分区内还会按照Key进行排序  
spark的shuffle结果还是分区有序,但分区内的Key无序  

要对spark的shuffle分区内再排序,有以下方法:  
* mapPartitions 再在每个分区上再使用.sort排序  
* repartitionAndSortWithinPartitions  重建分区,并排序  
* sortBy提前对RDD本身做一个全范围排序  

### RDD中会引起shuffle的操作  

* RDD重分区中,如果是调大分区,则必然使用shuffle  
* _ByKey之类的聚合操作,也会使用shuffle  
* RDD转换中,如果与父RDD采用不同的分区策略,则也会产生shuffle  


## 常用转换  

### Value数据类型  

#### 输入与输出一对一(窄依赖)   


**map系特点**  
> 默认保留以父RDD(*dependencies.head*)的分区,所以是一对一的窄依赖  
> 底层一般转化为 MapPartitionsRDD  


##### map  
元素一对一转换 所以子RDD的元素数量与父RDD一一对应  
```scala
val rdd = sc.parallelize(Seq("aa bb","cc dd","ee ff"),2)
rdd.map(rec=>rec.split(" ")>).collect().map(println(_))
//返回结果是rec.split(" ")结果(一维数组)=>[["aa","bb"],["cc","dd"]]
```

##### flatMap  
元素一对一转换为一个可迭代的元素,并将迭代元素扁平化  
```scala
val rdd = sc.parallelize(Seq("aa bb","cc"),2)
rdd.flatMap(rec=>rec.split(" ")).collect().map(println(_));
//返回结果["aa","bb","cc"]

//flatMap返回结果必须是TraversableOnce[U](可迭代一次的类型)
//所以如以下这种方式使用是不行
//rdd.flatMap(rec=>(rec,1)).collect().map(println(_));　　　
```

##### mapPartitions  
分区不变,对分区内元素进行迭代操作  
```scala
val rdd = sc.parallelize(Seq("aa bb","cc dd","ee ff"),2)
//mapPartitions 语法要求为 f: Iterator[T] => Iterator[U]
rdd.mapPartitions(part=>part.map(rec=>rec.split(" "))).collect().map(println(_))
//注意返回结果不是以分区为单位,而是所有分区执行函数的结果的合并
//所以结果是 [["aa","bb"],["cc","dd"],["ee","ff"]]
```

##### mapPartitionsWithIndex  
与mapPartitions类似,带有分区的index以供使用  
```scala
val rdd = sc.parallelize(Seq("aa bb","cc dd","ee ff"),2)
// mapPartitionsWithIndex 语法要求为 f: (Int, Iterator[T]) => Iterator[U]
rdd.mapPartitionsWithIndex((partIdx,part)=>part.map(rec=>(partIdx,rec))).collect().map(println(_))
//返回结果 (0,aa bb),(1,cc dd),(1,ee ff)
```

##### glom  
每个分区中的元素转换成Array[T],这样每个分区就只有一个数组元素Array[T]  
```scala
var rdd = sc.makeRDD(1 to 10,3)
//3个分区,glom结果3个Array[T] 每个Array[T]=分区下全部元素
rdd.glom().collect.map(x=>println(x))
```

#### 输入与输出 多对一 (窄依赖)  

##### union  
相同数据类型RDD进行合并，并不去重  

> 底层转为PartitionerAwareUnionRDD  

```scala
val rdd = sc.parallelize(1 to 10)
val rdd2 = sc.parallelize(5 to 15)
rdd.union(rdd2).map(rec=>rec.toString).collect().map(rec=>print(s"${rec} "))
//返回结果 1 2 3 4 5 6 7 8 9 10 5 6 7 8 9 10 11 12 13 14 15
```

##### intersection  
相同数据类型RDD进行合并,并去重  

```scala
val rdd = sc.parallelize(1 to 10)
val rdd2 = sc.parallelize(5 to 15)
rdd.intersection(rdd2).map(rec=>rec.toString).collect().map(rec=>print(s"${rec} "))
//输出 (1,2) (1,3) (1,4) (2,2) (3,2) (2,3) (2,4) (3,3) (3,4)
```

***cartesian*** 对RDD内的所有元素进行笛卡尔积操作   

> 底层转为CartesianRDD  

```scala
val rdd = sc.parallelize(1 to 3)
val rdd2 = sc.parallelize(2 to 4)
rdd.cartesian(rdd2).map(rec=>rec.toString).collect().map(rec=>print(s"${rec} "))
//输出 (1,2) (1,3) (1,4) (2,2) (3,2) (2,3) (2,4) (3,3) (3,4)
```

#### 输出是输入的子集(窄依赖)  

##### filter  
对RDD进行过滤操作   
##### distinct  
对RDD进行去重操作   
##### subtract  
RDD间进行减操作，去除相同数据元素   
##### sample & takeSample  
对RDD进行采样操作   


#### 输入与输出 多对多(宽依赖)  

##### groupBy   
将元素通过指定函数生成相应的Key,再按照Key进行分组  

> 如果是为了聚合而进行的分组,请使用 reduceByKey/aggregateByKey   
> groupBy 是纯粹的分组,因此无法在分组阶段提前聚合而导致会全量输出分组内容  
> 详细在后面 group 与 reduceByKey 对比时详细讨论  

```scala
val rdd = sc.parallelize(1 to 10,3).map(x=>x.toString)
//自定义按字符串 hashCode 分组
rdd.groupBy(x=>x.hashCode)
rdd.collect().map(rec=>print(s"${rec} "))
```



### Key-Value数据类型  

> ByKey底层会转化为PairRDDFunctions[K, V]  
```
/**
 * Extra functions available on RDDs of (key, value) pairs through an implicit conversion.
 */
class PairRDDFunctions[K, V](self: RDD[(K, V)])
    (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null)
  extends Logging with Serializable {
  ......
```

#### 一对一   

#### 聚合   

> 聚合都是多对多的宽依赖  


##### groupByKey   
按Key进行分组  

```scala
val rdd = sc.parallelize(Seq("aa bb","cc dd","bb cc"),2)
rdd
  .flatMap(rec=>rec.split(" ")).map(rec=>(rec,1))
  .groupByKey().map(x=>(x._1,x._2.sum))
  .collect().map(rec=>print(s"${rec} "))
//输出 (aa,1) (dd,1) (bb,2) (cc,2)
```

##### reduceByKey  
按Key进行分组后,用指定的函数对每个Key的所有Value进行聚合  

```scala
val rdd = sc.parallelize(Seq("aa bb","cc dd","bb cc"),2)
rdd
  .flatMap(rec=>rec.split(" ")).map(rec=>(rec,1))
  .reduceByKey((value1,value2)=>value1+value2)
  .collect().map(rec=>print(s"${rec} "))
//输出 (aa,1) (dd,1) (bb,2) (cc,2)
```

##### aggregateByKey    
给出一个默认基准值,先使用seqOp遍历分区内元素传入基准值进行聚合,再对分区间结果使用combOp聚合为最后结果  

```scala
  val rdd = sc.parallelize(Seq("aa bb", "cc dd", "bb cc"), 2)
  rdd
    .flatMap(rec => rec.split(" ")).map(rec => (rec, 1))
    .aggregateByKey(0)(
      seqOp = (u, t) => u + t,
      combOp = (u1, u2) => u1 + u2
    )
    .collect().map(rec => print(s"${rec} "))
  //输出 (aa,1) (dd,1) (bb,2) (cc,2)
```

##### combineByKey & combineByKeyWithClassTag   

(*combineByKey是对历史版本的兼容,1.6.0版本已全体更新为combineByKeyWithClassTag*)  
combineByKeyWithClassTag 算是一个比较核心的高级函数了.   
说高级是因为前面的*groupByKey,reduceByKey,aggregateByKey* 框架内部都会转调它  

来详细说说这个函数:  

1. combineByKeyWithClassTag 是属于PairRDDFunctions扩展的高级方法  
这意味着,只有[K,V]类型的RDD才有资格使用它  
2. 分区  
默认分区函数为 HashPartitioner,也可以通过参数传入自定义的Partitioner进行分区  
3. 语法结构  
```scala
def combineByKeyWithClassTag[C](
      createCombiner: V => C,
      mergeValue: (C, V) => C,
      mergeCombiners: (C, C) => C,
      partitioner: Partitioner,
      mapSideCombine: Boolean = true,
      serializer: Serializer = null)
```
* combineByKeyWithClassTag会遍历分区中所有的元素,遍历过程中每个元素的Key要么已存在,要么不存在
* createCombiner: V => C  
遍历过程中如果元素的Key不存在,就会使用createCombiner函数来创建一个初始的基准值  
* mergeValue: (C, V) => C  
遍历过程中如果元素的Key已存在 就会用mergeValue函数去聚合 当前基准值(C) 和 当前元素(V)  
* mergeCombiners: (C, C) => C  
合并各个分区并按Key分组后, 合并相同分组(相同Key)的基准值,成为最终结果  
4. WordCount 例子  

```scala
  type WordCount = (String, Int);
  rdd
    .flatMap(rec => rec.split(" ")).map(rec => (rec,rec))
    .combineByKeyWithClassTag(
      word => (word, 1),
      (c:WordCount, v: String) => (c._1, c._2 + 1),
      (c1:WordCount, c2:WordCount) => (c1._1, c1._2 + c2._2)
    )
    .collect().map(rec => print(s"${rec} "))
    //输出:(aa,(aa,1)) (dd,(dd,1)) (bb,(bb,2)) (cc,(cc,2))
```

#### 连接   

> 连接操作 即可能产生窄依赖 也可能产生宽依赖  
> 如果父RDD与子RDD的分区算法相同(比如都是默认的hash-partitioned) ,则最终产生窄依赖  
> 如果父RDD与子RDD的分区算法不同,则需要按照新算法产生shuffle重分区,最终产生宽依赖  

##### join  
根据两个RDD的Key进行关联,没有关联上的数据将会丢失  


##### leftOutJoin & rightOutJoin  
左外&右外关联.以左RDD(右RDD)为主进行关联,在另一边为匹配的将填充空值  


##### fullOuterJoin  
两个RDD 关联后的笛卡尔积  

##### cogroup  
将多个键值对RDD按Key合并在一起.合并为全数据(没有丢失)  
与fullOuterJoin区别:多个RDD情况下,cogroup按key合并为一个,fullOuterJoin为多个的笛卡尔积  
注意,如果某个数据集少某一个key,合并时是在这个数据集的位置上占CompactBuffer()的位置,而不是直接跳过  

```scala
val rdd = sc.parallelize(Seq("a b")).flatMap(rec => rec.split(" ")).map(rec => (rec, rec));
val rdd2 = sc.parallelize(Seq("b c")).flatMap(rec => rec.split(" ")).map(rec => (rec, rec));
rdd.cogroup(rdd2).collect().map(rec => print(s"${rec} |"))
//(b,(CompactBuffer(b),CompactBuffer(b))) |(a,(CompactBuffer(a),CompactBuffer())) |(c,(CompactBuffer(),CompactBuffer(c))) |
```

**cogroup是RDD所有连接操作的基石**  
> RDD的大部分连接操作底层都是用过 cogroup 实现的  
> 这意味着,其实都是先按Key合并为全数据,再按照不同的连接条件进行过滤  
> join 过滤保留*结果左右值* 都存在的  
> leftOuterJoin(rightOutJoin) 过滤保留 *结果左(右)值*存在的  
> fullOuterJoin 过滤保留*结果左右值* 都存在的,然后做一次笛卡尔积组合

##### subtractByKey  
根据两个RDD的Key进行关联,然后去除已关联上的只保留未关联上的

---

#### 重要 

##### groupByKey 与 reduceByKey 区别  

groupByKey 与 reduceByKey ,本身是不一样的.一个是分组,一个是分组聚合.  
但如果用groupByKey来完成分组+聚合,就可能会有一定的性能问题.  

以两个最后结果完全一致的用法举例  

分组聚合  wordsRDD.reduceByKey(_ + _)   
> 因为已经知道了聚合逻辑.所以会在shuffle分组阶段提前进行一次聚合(*combine*)   
> 这个提前聚合对性能提升非常大,因为它会大幅减少map的输出  
> 比如 word的1千条记录,如果提前聚合最终map输出的只会是1条 (word,1000)  

分组+聚合 wordsRDD.groupByKey().map(t => (t._1, t._2.sum))  
> 分组和聚合是拆成两个独立的操作. 这导致分组时因为不知道聚合逻辑而无法进行map聚合.  
> 所以分组的map输出是全部输出.这会带来高IO的性能损失


### 分区调整

#### coalesce   
重新调整RDD的分区后形成一个新的RDD.  
语法要求:numPartitions: Int, shuffle: Boolean = false  
* numPartitions表示要重新调整的分区数量
* shuffle表示重新调整分区时是否允许发生shuffle过程

#### coalesce调整分区与shuffle  

* 如果子分区数往下减少,则子分区数设置一定会成功.  
但要注意,在这种情况下会造成任务的并行度降低(分区数,任务数降了),任务内存更容易爆出(单个任务的数据增大了)  
* 如果子分区数往上增加,则子分区数设置必须要设置shuffle=true,才会成功,否则子分区依然等于父分区   

重分区不一定会产生shuffle.比如减少分区的窄依赖  但是如果是增大分区,则必须依靠shuffle  
谨记原则:**如果没有shuffle的参与,RDD只能减少分区(窄依赖),不能增加分区**  

#### repartition  
只是coalesce的shuffle等于true的快捷方式. coalesce(numPartitions, shuffle = true)  


