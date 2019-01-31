---
layout: post
title:  "Streaming架构"
date:   2018-10-16 13:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}


## 概述  

Streaming 是Spark的流式应用,是整体架构在Core上的上层应用  
Streaming 是一个微批实时流,本质上就是将实时进入的数据按时间维度分批计算 

##   编程模型  

### DStream   

DStream 是Spark对实时数据流的核心抽象  
Streaming整体是架构在SparkCore之上的,所以DStream的本质是一个以时间为键,多个批次RDD数据组成的`HashMap[Time,RDD]`  
* 在Spark之外的而言,数据会持续不断进入,所以呈现出无界流的特征.(事实上所有的实时流抽象都必然是数据无限进入,从而呈现无界流的特征),而在DStream中随着时间流逝,不断有批次数据(RDD),加入到`DStream.HashMap`中,而这是可以无限扩容的  
* 事实上无论是出于内存或存储上限还是计算能力上限,都不可能出现真正的无界流.在Spark内部而言,体现在随着不断的处理,批次数据也不断的从`DStream`中被移除.对于无界流中的历史,Spark采用的是`状态流`解决方案.这里的是状态,指的是`历史数据的汇总结果`,所以DStream的无界流,事实上是以`有界流(当前数据流)`+`状态流(历史汇总结果)`合力完成的  
* 进入和移除的核心维度,就是HashMap的键`Time`  

### DStream的具体编程模型  

DStream本身是一个抽象概念,它会继续细化为以下三个具体偏向编程模型  
* InputDStream 数据源(采集)   
* TransformDStream 转换计算  
* OutputDStream 执行输出  

## 架构  

### DStreamGraph  

`DStreamGraph` 是Streaming的计算中心,有两个核心数据结构(本质就是对数据部分执行计算)  
* `inputDStream:ArrayBuffer[InputDStream]`  数据部分  
数据不断的收集汇集到inputDStream中  
* `outputDStream:ArrayBuffer[DStream]` 计算(执行输出部分)  
定时触发对inputDStream执行一系列的计算和输出  

### 数据体系  

数据的收集,首先是开始于一个`InputDStream`,而每一个`InputDStream`启动时首先会将自身注册到`DStreamGraph`中   

#### 数据收集  

数据收集首先是构建收集体系.这个采集体系是一个主从架构  
* **主**  `ReceiverTracker`  
运行在driver端,作为采集中心点和元数据管理  
* **从** `ReceiverVisor`  
运行在executor端,对上向`ReceiverTracker`注册,对下管理`Receiver`  
* `Receiver`  
是`ReceiverVisor`之下的线程概念,负责真正的数据收集工作  
在Spark1.5之后Spark开始视`Receiver`为一个独立的`Task`(即由Spark分配一个独立Task来执行Receiver),这样做的好处是方便Spark做高可用,Spark是以Task为单位调度执行,如果某一个`Receiver`(`Task`)有问题或者崩溃,只需要直接杀死然后另找一个executor分配重启这个Task就行了  

在Streaming启动后,driver通过ReceiverTracker向所有ReceiverVisor发出启动命令,ReceiverVisor收到启动命令后,会启动自身所有的Receiver线程,开始收集过程  

#### 数据汇集  

Receiver收集数据后会写入`ReceiveBlock`(等同Core中Block),并通知自己的ReceiverVisor,ReceiverVisor则会将该`ReceiveBlock`元数据上报给ReceiverVTracker  
这样数据写入`ReceiverBlock`,但元数据全部汇集到driver的`ReceiverTracker`中  

#### 预写日志  

流式应用与离线分析不同,难以强求一个有力的可靠数据源保证,其数据源往往不可回溯,虽然Streaming有数据缓存机制,但一旦executor崩溃,数据就很难恢复了  
为了应对这种情况,Streaming加入了`预写日志`机制  
Receiver收集写入`ReceiverBlock`中后,会同时写入到一个第三方的可靠文件系统中(`HDFS`),这样一旦executor崩溃可以从文件系统中恢复数据  
* `预写日志`的优势在于数据0丢失  
* `预写日志`的劣势在于绝大部分情况下,文件系统中数据都是浪费的,而且写入文件对Streaming的运行效率牺牲非常大  
因此,预写日志一般只是一种无奈的选择,更好的解决方案依然是尽可能寻求可靠数据源保证(`Kafka`)  

### 计算体系  

#### 注册执行  

DStream中`输出执行(output)`(等同Core中Action),会将这个DStream转换为一个OutputDStream,并将其注册到`DStreamGraph.outputDStream`中  
这里与Core不同是DStream.Action仅仅是注册执行而不是真正触发执行,因为DStream的输出执行还有一个时间维度,所以仅仅代表将来某时执行  

#### 触发执行  

`输出执行(output)`的真正执行依赖`JobGenerate`
`JobGenerate` 负责Streaming计算任务生成,本质上就是一个定时器,这个定时器的执行间隔就是`StreamingContext`里设置的间隔  
定时器的每一次执行触发就是触发一轮Streaming的批次计算,具体过程如下  
* 触发`DStreamGraph`的任务生成  
对`DStreamGraph`中的outputDStream遍历每一个执行`outputDStream.generateJob`
* `outputDStream.generateJob`是对其内部的所有RDDAction包装成Job=>Seq[Job]  
* 将`Seq[Job]`(计算逻辑),`ReceiverBlock`(数据部分,来自InputDStream),`Time`时间维度三部分共同包装成`JobSet`,然后交给Core引擎执行  

