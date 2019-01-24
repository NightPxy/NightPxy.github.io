---
layout: post
title:  "Spark-Shuffle(未完成)"
date:   2018-06-11 13:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}


## 概述  

Spark的Shuffle混洗本质上就是同Key分组,是将散落在不同原始分区(Partition)的同Key数据按照Key进行重分组,其原理和思想与Hadoop的Shuffle完全相同  

Spark的Shuffle在Hadoop基础上有了一些演进  
Hadoop的Shuffle主要是Sort-Shuffle思路.即在Map阶段将每个分区数据按照Key排序写入  
Spark的Shuffle主要有以下演进过程  
* Hash-Shuffle  
* Sort-Shuffle  
* Sort-ByPass-Shuffle  
* OffHeap-Shuffle   

## shuffle代价  



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

因为Hash-Shuffle的恐怖文件句柄问题,Spark最终再一次回到了Hadoop的解决方案:Sort-Shuffle  

### 核心思路  

整个Map端的Shuffle过程就是一个LSM算法  
* Map阶段分块分段(受内存限制)读取,溢写文件到磁盘(溢写文件内保持Key有序)  
* 多个溢写文件归并排序为一个最终文件,依然保持有序  
同时为这一个文件生成Index文件,标记这个文件的Key分布(每个Key的起始和结束Offset)  
* 每个Reduce读取时通过Index从最终文件中拉取数据  

### 优势  

解决了文件数过多的问题,文件数为 `Map数量`  

### 劣势  

倒回到以前的排序消耗  

### Sort-Shuffle优化


#### bypass机制  

bypass机制类似与这两者的结合版  
* 首先沿用Hash思路,按Key分组不排序  
* Key分组后会合并写入最终文件,而不是每个Key保留一个文件  
* 沿用Sort-Index思路,方便Reduce拉取  

bypass的优势  
一定程度上解决了Sort与Hash各自的弊端  

byass的劣势(也是ByPass的限定条件)  
* Map数量在`spark.shuffle.sort.bypassMergeThreshold`阈值(默认200)以内  
* 非聚合类算子(`reduceByKey`)  
bypass机制是直接合并,没有combine操作,不适宜用在聚合算子中  

