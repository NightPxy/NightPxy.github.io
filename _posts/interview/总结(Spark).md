---
layout: post
title:  "Spark总结"
date:   2018-09-01 13:31:01 +0800
categories: interview
tag: spark
---

* content
{:toc}


# Spark  

## RDD的分布式弹性数据集体现  

在我的理解中,RDD的弹性主要体现在对数据集的抽象封装上.也就是指RDD描述的是一个抽象数据集而不是物理数据集.在这种抽象层面,为RDD带来容错,分区变换,自动内存与磁盘的切换等等  
首先,Spark在RDD中抽象出了依赖关系.每一个RDD在产生时都会记录产生它的父RDD(或者是顶层数据源),以及如何从父RDD转换到自己的过程.这样在某一个RDD出现错误时,可以从父RDD在此计算从而重新恢复  
其次,RDD封装了数据的存储.使我们不关心数据本身是来自内存或者磁盘,从而使数据集本身可以在内存和磁盘中切换  
再次,RDD透传分区,而使我们在使用时可以不关心数据集本身的分布式分布情况  

## RDD的五大特性以及体现  

* 分区
RDD的分区是一个逻辑概念.两个RDD的分区在物理上可能是同一个内存中的同一个对象,而不一定是完全独立的.这是一种故意破坏函数不变性的实现,也是为了防止因为函数不变性带来的对象膨胀  
* 按时

## 常用RDD简述  

## RDD的分区数计算    
RDD的分区数大致计算方式如下:
* 在通过本地数据集合创建的RDD中,分区数量等于spark.default.parallelism 默认情况下本地模式下等于核数,YARN模式下等于executor数量(或者2,谁大取谁)  
* 在HDFS之类的外部文件中,分区数等于文件块数,即一个文件块对应一个分区  
此时HDFS的文件块不能超过128M(spark.files.maxPartitions) ,另Spark内置最小分区为2,即如果文件只有一个块,也会是两个分区 ???源码验证 
* 在KafkaDirect中,分区数等于kafka的分区数,即一个kafka的分区对应一个RDD的分区 
* 在HBase上, 分区数等于HBase的Region数  

## RDD的缓存是立即进行吗,缓存级别有哪些以及如何选择  

首先RDD的缓存是Lazy的,也就是调用persist只是为RDD打上一个标记,并不会产生实际执行,缓存的执行必须通过Action来完成  
因为缓存的实际执行是在RDD的遍历方法中,首先通过这个标记,透过BlockManager去尝试读取存储.如果读取成功就会就此返回,如果读取失败才会调用RDD计算方法进行计算,并将计算结果再次写入.而RDD本身是Lazy的,触发RDD遍历方法的先决是执行Action.所以缓存也必须通过Action  

Spark提供的缓存级别有仅内存,内存且允许溢写到磁盘这三种,配合是否序列化6种,配合更多副本数更多.在选择上,官方的建议是如果仅内存没有问题那就仅内存最好,如果频繁发生GC或者OOM,可以尝试使用序列化内存,并且是kyio的序列化内存,可以减少缓存内存占用,当然代价是消耗更多的CPU资源来进行序列化和反序列化.除非是那种重新计算比从磁盘读取慢的多的情况,否则一般情况不考虑磁盘.

## RDD结构以及源码简述  

Spark对RDD设计呈现设计模式中的模板模式  
即在RDD的抽象类中,Spark为RDD封装了大量的通用逻辑,再某些RDD细节交由子类实现
整个RDD的使用都是由抽象类提供模板实现,比如compute,在RDD之外是不使用的,一个RDD的分区计算的真正展开是抽象类的iterator方法  

交由子类实现中,最核心的部分包括两个: compute(分区计算逻辑) 和 getPartitions(分区列表)  
并对诸如partitioner(分区器),提供默认实现None,即视为HashPartition  

而iterator,个人理解中RDD中最重要的是iterator.这是RDD的真正对外实现.内置会判断RDD带是否带缓存级别,如果有就会先从缓存中尝试读取,只有读取失败才会调用子类细节实现的compute方法,并同时会将计算结果再次尝试缓存  

RDD的五大特性,都是由RDD抽象类提供  




此外RDD的抽象类还会提供一些通用算子比如map,flatMap等


* RDD的构造需传入SparkContext和 Seq[Dependency[_]]  

* 所有RDD都自动继承Serializable

## BlockManager怎么管理硬盘和内存的  

Spark通过BlockManger,在Driver和Executor之间构建出小型的分布式存储系统,并以此为依托构建的整个Block存储体系  

**Block为最小存储单位**
Spark抽象的最小存储单位为Block(不存在半块).一个Block等价于RDD一个分区,每一个Block都会有一个唯一ID  

**分布式存储网络**
在Spark启动时会在SparkContext中通过BlockManager分别在Driver启动BlockManagerMaster,在Executor启动BlockManagerSlaver,Master负责管理Block元数据并对外暴露元数据服务,Slave通过Netty在Master完成注册并维持心跳,并对外暴露本节点的数据传输服务.这样首先构建出类似NN和DN分布式存储通讯架构  

**元数据管理**
BlockManagerMaster类似NN,负责管理Block元数据.其中是通过BlockManagerMasterEndpoint内存维护的三个HashMap来负责存储元数据   

*  ExecutorId与BlockManagerId (某一个Executor对应的BlockManager是谁)   
*  BlockManagerId与BlockManagerInfo(BlockManager的已存储块信息,目标传输服务引用等)  
*  BlockId与BlockManagerId集合(表示块被存储在哪些BlockManager节点上,多个表示副本情况)

**读取**
在读取时,首先通过getLocation方法,通过BlockId请求Master的元数据服务,获取Block的元数据,这些元数据包括存储方式(内存&磁盘),存储的Slaves位置以及副本数等等  
*  通过元数据可以感知Block是以内存还是磁盘存储,并同时对返回位置信息按照数据本地性依次请求.
*  如果是本地读取,通过getLocalValue读取.读取时会根据是内存还是磁盘切换MemoryStore或DiskStore来负责真正读取.MemoryStore的读取就是直接内存读取,因为Spark的内存存储载体是一个LinkHashMap.DiskStore则通过文件名计算出文件块的本地磁盘路径读取  
* 如果是远程读取则getRemoteValues方法,其通过元数据返回的位置请求目标BlockManagerSlaver暴露的数据传输服务拉取  

**写入**
在写入时,也会请求Master的元数据服务来获取需要存储方式,存储节点以及副本数等等  
* 如果是内存写入,就会相当于写入LinkHashMap.此时会将所有的Block逐块写入,每写入之前会查看内存标量是否足够,如果足够则直接写入并提升内存标量,如果不够则会尝试扩容,直到超过当前Executor的存储内存则会弹出,以及设定存储方式决定是溢写到磁盘还是就此放弃  
* 如果是磁盘写入,就会根据文件名取两次Hash分别作为一级目录(16位UUID)和二级目录(64)并组装成本地磁盘路径,然后写入  
* 如果需要写入副本,则会联系副本目标所在的BlockManagerSlaver的传输服务写入  
* 全部写入完毕后,再次联系Master元数据服务告知写入完毕  

## RDD的宽依赖与窄依赖,有什么区别,Join是宽还是窄依赖  

宽依赖与窄依赖,是描述两个有血缘关系的RDD的分区之间的关系的描述  
如果父RDD的某一个分区全部进入子RDD的某一个分区,比如一对一或者多对一,就是窄依赖  
如果父RDD的某一个分区只能部分进入子RDD的某一个分区,就是宽依赖   

本质上,宽窄依赖其实对shuffle的描述.即事实上,宽窄依赖影响的是DAGScheduler绘制DAG图时是产生ResultStage还是ShuffleStage,并由此产生ResultTask还是ShuffleTask   

从执行效率来说,窄依赖有非常明显的优势,因为Spark的Stage才提交时会通过检查是有MissStage在执行而不断递归自旋,换句话说Spark的Stage是串行执行的,前一个执行完毕才会启动执行下一个.
这代表如果是纯窄依赖的ResultStage内部的Task将会并行执行,而宽依赖因为会被迫产生一个ShuffleStage,会先行执行ShuffleStage,并且在等待ShuffleStage所有Task全部执行完毕后,才会再提交ShuffleStage之后我们真正的FinalStage才会被提交执行.

## Spark是如何分解出DAG图的  

首先,绘制DAG的核心类是DAGScheduler  
绘制过程从Action开始,准确说是从每一个Action里的sc.runJob方法开始  
* sc.runJob 转到 dagScheduler.runJob,DAGScheduler正式接管  
* dagScheduler.runJob.submitJob 提交Job 
dagScheduler.runJob通过会将action的算子记录,然后连同action所在的RDD一起包装为一个Job.(这就是每一个Action产生一个Job)  
Job信息会提交到Spark内置的事件总线JobSubmit中,,并在handlerJobSubmit中获得实际处理  
* dagScheduler.handleJobSubmitted将Job(等价Action),分解Stage   
submitJob处理首先为这个Job构建一个FinalStage(ResultStage)(每一个Job包含至少一个Stage),然后沿着Job提交的RDD的血缘关系逆流而上进行溯源   
溯源过程中检查是否有shuffle(标准是判断RDD的Dependency是否是ShuffleDependency)  
如果没有shuffle就会加入当前Stage并继续溯源  
如果有shuffle,就会构造出一个ShuffleStage,然后再继续溯源.  
在Stage内部的溯源过程中按照广度优先的策略,将RDD不断压入一个ArrayStack并最终在溯源完毕后栈出完成倒装  
这样最终遍历结果就是这样一个(画图)Stages.  
每一个Stage被类型标注为Stage子类ResultStage和ShuffleStage  
每一个Stage中包含一组funs,这个funs的本质是这个Stage中的RDD的顺序排列的遍历函数.  
每一个Stage可能会包含0或多个父Stage,这表示Stage之间呈现依赖关系  
DAG绘制的最终结果就是这个Stages.Stages的本质是一组一组的组合好的funs.组内funcs呈现一对一或多对一的顺序过程,而组与组之间又呈现多对多的顺序过程.其数据结构就呈现出有向无环图的特征.这就是大名鼎鼎的DAG图 .有向有序的是从一个或多个Stage到另一个或多个Stage,而每一个Stage中又有序包含一个RDD到另一个RDD的调用方向,无环是这个调用方向是单向,不存在循环引用.  

## Spark的DAG图是如何调度执行的  

DAG图本质是有Stages组成的StageGroup.而在调度执行这个StageGroup是一个一个Stage串行执行,也就是执行前首先判断是否有MissStage正在执行并不断自旋递归.  
而调度执行Stage的核心类时TaskScheduler.而整个调度执行过程,非常类似YARN  

* 提前架构类似YARN的调度通讯网络  
首先在SparkContext时,就会启动ScheduleBackEnd线程(driver中),类似YARN中ResourceManager,并在executor中同时启动ScheduleExecutorBackEnd,并通过RPC向driver注册自己并维持心跳,driver会持有executor的RPC引用来获得对executor的通讯能力.这样首先在driver和executor之间建立一个主从的类似YARN中ResourceManager和NodeManager的调度通讯体系  

* 在dagScheduler.submitMissStage中解析stage  
首先获取出RDD的分区数量,这是后面决定生成多少个Task(一个分区对应一个Task)  
其次是获取出RDD的位置信息,这是后面任务调度时数据本地化的依据  
然后是序列化stage的执行过程.所谓的执行过程,就是RDD的遍历函数的反射位置.所谓的分布式执行,就是提前传输jar到executor,并依次反射出这些RDD遍历函数来执行  
最后就是根据分区数量构建出Tasks,将分区,位置,序列化的执行过程的相关信息等包装为一个Task,如果是ResultStage,就会包装成ResultTask,如果是ShuffleStage就会包装ShuffleTask  
最终的结果是将Stage最终分解为一堆设定好的TaskSet提交给TaskScheduler.submitTask调度执行(用Set包装,表示在Spark的视角中,不在乎Task的执行顺序,事实上在分布式环境上受限目标机的线程调度情况,也不可能保证Task的执行顺序)  

* TaskScheduler.submitTask接受TaskSet   
TaskScheduler.submitTask会遍历TaskSet调度执行每一个Task  
在每一个Task的调度执行过程中  
首先会构建TaskSetManager的调度器(FIFO或者Fair调度器)来调度执行这批Task  
其次会通过SchedulerBackEnd.reviveOffers发生TaskSet,在makeOffers中挑选节点执行  

* SchedulerBackEnd.makeOffers挑选节点执行
首先会获取当前可用的executor列表,并遍历TaskSet,将每一个Task和当前所有可用的Executor包装成一个WordOffer
再对每一个WordOffer,也就是每一个Task和所有可用的Executor列表之间进行过滤选择.  
过滤选择方法是首先排除黑名单Executor,再按照executor和host排序逐个尝试去占内存和核,目标是期望选用最接近数据的executor执行Task.一旦占用成功就选择该Executor为执行节点
最后通过TaskScheduler.launchTasks提交这批WorkOffer.将Task信息和将要执行这个Task的Executor一起包装为一个TaskDescription,这个TaskDescription最后将含有TaskId,ExecutorId,序列化的执行逻辑以及相关的AddJars和AddFile  
最后这个TaskDescription会由TaskScheduleBackEnd根据ExecutorId发送给目标Executor  

* Executor接受执行  
Executor在收到TaskDescription之后,通过launchTask解析,然后包装成TaskRunner的Runable,并提交给自己的Executor执行  

* 执行完毕后,根据Task任务类型决定如何返回  
如果是ResultTask,就会通过RPC告知driver,并计算结果发回  
如果是ShuffleTask,就会将Shuffle结果存入自己的BlockManager,并将BlockId等存储信息发回driver

## Spark的内存管理  

这里只针对executor的内存管理简述.  
在executor中,整个JVM的内存被隐式的分为三大块.  
* 预留内存  
预留内存,固定300M,起预留作用  
因为本质上,没有任何办法可以控制对象在JVM中实际的内存分布,所以所谓的分块本质是一种上限标量,即在整个JVM内存的总大小上设定一个各块使用的上限来完成.本质上说,无论用户,计算还是存储的对象都是在JVM堆上的对象,并没有本质区别.这里可能是Spark为了避免一些临界问题,所以引入了预留内存,理论上不在各块之中,但实际可以被用户占用.  
需要注意的是,源码中明确指定了启动Spark内存至少是1.5倍(源码写死的)预留内存,即Spark启动的最小内存是450M  

* Spark内存   
Spark框架所使用的部分,由`spark.memory.fraction`控制,默认0.6,即(JVM内存-预留内存)*0.6为Spark使用的上限  
Spark使用的内存,再继续分为两大块  **执行内存** 和 **存储内存**  
存储内存为shuffle结果数据存储,缓存以及广播变量等所使用的内存  
执行内存为shuffle中间数据存储,连接以及排序等Spark内置环节计算所使用的内存  
其计算比例为 存储内存`spark.memory.storageFaction`控制,默认0.5,剩余为执行内存  
这其中存储内存和执行内存可以相互借用,规则如下:  
存储内存可以超出执行内存的阈值来进行存储,但当执行内存需要时可以立即释放多余存储  
执行内存可以超出存储内存的阈值设定0.5来进行计算,但可以为存储内存设定一个最小保留值,但存储需要时执行内存不还,而是等待执行结束  

简单来说,就是存储借用必还,执行借用不还.因为存储的占用是按块为单位,基于RDD容错机制,直接干掉无所谓,但执行内存的计算上下文非常复杂,所以正在执行的计算内存不予归还  


* 用户内存  
用户内存是指在用户使用的内存,包括用户在算子使用的一些临时数据或者数据结果等等  
比如mapPartition算子,将返回分区下全部数据列表,这个List数据结构将占有用户内存  

* 2G内存计算如下  
预留内存:300M  
Spark执行内存: (2048-300)*0.6*0.5 = 524   
Spark计算内存为: (2048-300)*0.6* (1-0.5)=524   
Spark用户内存为:(2048-300)*0.4=699  

## Spark的OOM一般有哪些,如何处理  

OOM即内存溢出.简单来说,就是JVM上的堆满了,GC清理掉不可用的对象之后还是分配不了内存,就会触发内存溢出  
Spark本质上是有driver和executor两种JVM,所以首先区分的是OOM是driver溢出还是executor溢出  

**driver溢出**  

* 首先要关注的是executor的结果返回接收  
一般情况下,不应该返回executor的计算结果,尽量在executor解决,这也更符合Spark设计的初衷,就算有返还结果,也应该是非常少量的,比如抽样100条这种  
如果涉及大结果返还,一个driver或面对N多的executor,很容易撑爆driver.这里就必须调整逻辑,尽量在executor完成逻辑  
* 其次是driver的内存是否过小  
driver将作为整个Spark作业的计算中心  
包括BlockManager的元数据存储,TaskScheduler的调用监控以及executor的元数据,StageSchedule的整个DAG分解图等等,所以承载的东西非常多,所以相对的需要将driver资源适当提高(NN和RM,也会要求是集群能达到的最好的配置不是)  

**executor溢出**  

* 首先应该为executor分配的内存大小  
这里的合适的标准在我的理解非常简单,就是一个executor的内存必须至少能保证在完整放下整个分区数据,并且还能有余力稍微做一点事情  
完整放下分区,与优化无关.因为这不是我们能代码优化解决的,再怎么说,算子总要用吧.  
在Spark中,相当多的算子在设计之初就是需要加载整个分区,比如mapPartition,这也是Spark高吃内存的原因之一,其底层就决定了必须能承载一个分区  
* executor必须保证容纳分区的同时还能有一点余力做事,但是受限集群资源,有时有不能做到这一点,也就是executor的资源有上限,也就是加资源不行,此时另一个办法就是减分区大小.  
减分区的大小非常简单,就是重分区.reparation,通过提升分区的数量来减少分区的大小.  
分区小了,也就能满足executor完整容纳的要求  
但是通过reparation减分区不是万能的,这里有涉及倾斜的问题    
* 倾斜问题  
倾斜问题,可以简单看作同Key问题.  
同Key倾斜非常容易出问题,可能引发shuffle溢出,shuffle不溢出也可能导致分区数据过大也产生溢出. 
因为分区的依据是Key,分区的本质是通过Key的Hash对分区数取模来落入分区,这有个问题是如果是同一个Key,无论怎么调整分区数量,都不会减少分区大小,因为同Key必然同分区  
如果有同Key倾斜,最简单的办法就是打散分区,比如为Key添加随机缀等,通过改变Key来打散分区  
注意打散时需要注意是否还需要再聚合,如果是不需要聚合怎么打散都无所谓,如果是需要聚合必须按照一定的格式打缀,这样通过打散之后的聚合结果再去缀进行二次聚合    
*  代码逻辑   
本质上,executor就是一个java应用,跑在JVM.没有任何区别,所以一些原来的代码优化也是可行的  
比如循环内的反复创建等等,比如map算子的循环new StringBuilder(线程不安全)等等    
* coalesce  
这是一个容易忽视的问题.  
假设原先有100个块,应该是100个task,但是因为其结果输出过少,为了防止小文件coalesce(10),结果内存爆了.  
coalesce的调小分区是窄依赖,不会产生shuffle.所以coalesce(10)设定RDD的分区数是10,会认为stage的分区是10,taskScheduler会因此认为应该使用10个task.原来100个task变成10个task,一个task读10块就爆了  
此时需要使用coalesce强制使用shuffle为true或者直接使用reparation来降分区,通过shuffle为打断stage,前面有100个task读,计算结构再交由10个task写出去  

coalesce(numPartitions: Int, shuffle: Boolean = false)
repartition=coalesce(numPartitions, shuffle = true)  

## 哪些算子会有Shuffle  

判断shuffle的标准是  
RDD的Dependency是ShuffleDependency.如果是就会在Stage分解中产生ShuffleStage  
如果是NarrowDependency或者OneToOneDependency 就是窄依赖  

比如ShuffledRDD,凡是转换为ShuffledRDD的都会产生依赖,内置依赖就是ShuffleDependency  
用到ShuffleRDD的算子,就会产生Shuffle.比如*ByKey都会产生Shuffle,因为所有*ByKey都是combineByKeyWithClassTag的重载调用,而combineByKeyWithClassTag内部就是转换ShuffleRDD.还有coalesce在设定为shuffle为true也是,同理可知reparation也会产生shuffle.  
还有join,join是可能产生shuffle可能不产生.因为其依赖是一个if判断.如果join之间保持分区协同就是OneToOneDependency ,此时不会产生shuffle.如果不能保持分区协同就是ShuffleDependency,就会产生shuffle  

## Streaming  

Streaming是构建在RDD上的实时流处理.  
Streaming通过接受实时流数据,并按照一定的间隔拆分成一批一批的数据,或按照一定的间隔去定时的拉取成一批一批的数据,来组成一个一个RDD  
按时间间隔产生的一个一个RDD,通过一个先进先出的队列生产出去,并最终通过Core来消费这些RDD并最终输出,本质上一个生产消费者模型  

在Spark之外看,无论是Spark接受数据拆分成批次,还是按照间隔定时拉取成批次,看起来都是一个持续不断的数据流和处理过程.这就是Streaming对流的认知,这是一个持续性的数据流,并抽象成编程模型DStream  
在Spark之内,这个DSream是一个由一个或多个RDD组成的集合.其本质是一个以时间为键,RDD的值HashMap.随着时间的流逝RDD会不断的进入这个HashMap,而随着我们的处理又会不断的从这个HashMap移出.而所有的操作本质上都是对这些RDD操作,DStream是在RDD之上的再次封装而已

所以DStream在设计上是一个没有边界没有大小限制的无界流,随着时间的推移会不断的产生RDD又会不断的消耗RDD 

DStream是一个RDD DAG上增加了时间维度.一个Action在RDD的角度上讲是执行,但是在DStream的角度却不能这么说,而是说会在时间片去执行  

Streaming把根据事件流逝进入的数据划分成不同的job,每个Job都有对应的RDD依赖,而每个RDD都依赖输入的数据,所以这里看成不同RDD依赖构成的批次,而这个批次就是Job,然后得出一个又一个结果  
以时间为批次划分Job,就好像一个巨大的定时器.定时执行我们的Action(Job)
所以DStreamGraph不断生成RDDGraph,就好像Core不断调用Action一样产生Job  


输出重复问题  
Core天生会做以下事情而导致任务重新执行,以致输出重复  
Task重试  
慢任务推测执行  
Stage重试  
Job重试  

设置spark.task.maxFailures次数为1，这样就不会有Task重试了。
设置spark.speculation为关闭状态，就不会有慢任务推测了，因为慢任务推测非常消耗性能，所以关闭后可以显著提高Spark Streaming处理性能  
Spark Streaming On Kafka的话，Job失败后可以设置Kafka的参数auto.offset.reset为largest方式  
但是这样处理是非常有问题的,也不应该去从这些角度去解决重试问题.最终极的办法应该是想办法做输出幂等,即让重复输出无害  

https://blog.csdn.net/csdn_zf/article/details/51346283 

## Streaming 与 DStream    

Streaming是Spark的流式处理应用,是整体架构在Core上的上层应用  
在Streaming中,以DStream为核心编程模型,以JobScheduler为核心计算体系  

DStream,代表着Streaming对数据流的抽象.但因为Streaming构建在Core上,所以DStream的本质是一个以时间为键,多个RDD组成的HashMap[Time,RDD],随着时间流逝不断有RDD进入,同时随着Streaming对数据的处理,也不断的有RDD被移出,而进入和处理的核心维度就是Time     

**无界流与有界流  **
在Streaming之外的视角,数据会随着Streaming应用的启动而不断进入.事实上对于所有的流式数据而言,数据都是以连续不断的形式不断的涌入,所以DStream首先呈现出无界流的特征,那就是可以包容所有进入的数据,也就是HashMap[Time,RDD]可以无限扩容  
但事实上,无论是内存上限还是计算效率,都不可能存在真正意义上的无界流,所以DStream的是随着计算处理完成而移出RDD的,而对于无界流的历史功能,Streaming引入的是状态流概念,这里的状态,指的是历史数据的聚合结果,所以在DStream的无界流,事实上是历史状态聚合+当前数据流  

**DStream具体的三个编程模型  **
* InputDStream  
* TransformDStream  
* OutputDStream  
这三个模型与Core有些类似,分别是数据采集(数据源),转换计算和执行(输出)  

## Streaming 的计算架构  

Streaming的计算架构依赖两个核心组件,  DStreamGraph 和 JobScheduler  

**DStreamGraph**
DStreamGraph,是作为Streaming的计算中心存在,它随着StreamContext的启动而启动,内部有个两个关键的数据结构 inputDStream:ArrayBuffer[InputDStream]和outputDStream[DStream],分别代表Streaming的数据部分和计算部分.Streaming的计算本质就是对数据部分进行计算部分  


JobScheduler负责DStream数据的产生和计算产生 ,其内再包含两个核心组件ReceiverTracker和JobGenerator  

**数据体系**  
数据的收集,首先是一个InputDStream.每一个InputDStream在构造时就会注册到DStreamGraph的inputStreams.而InputDStream作为一个DStream,其本质是一堆RDD,而RDD的本质是分布式数据块,所以整个数据采集体系的本质是,采集数据汇聚到数据块ReceiveBlock中,并将ReceiveBlock填充到InputDStream的RDD中    
其整个体系由ReceiverTracker,ReceiverSupervisor和Receiver共同完成  
* 首先是构建数据收集体系  
数据采集体系,也是一个主从架构,主是ReceiverTracker,运行在Driver端,作为数据采集的中心点和ReceiverBlock的元数据中心,从是Receivervisor,运行在Executor端,向ReceiverTracker完成注册并负责管理Receiver,而在Receivervisor之下的Receiver线程,就是真正收集过程,这个收集过程由InputDStream的getReceiver给出执行过程   
随着Streaming.start,ReceiverTracker会向所有Receivervisor发出启动命令,Receivervisor收到启动命令后启动自己管理的Receiver线程  
* ReceiveBlock汇聚过程  
Receiver接收或拉取到数据会,会写入到ReceiveBlock(等同Core的缓存,只是在Streaming中,默认缓存级别是内存或磁盘)并通知自己的Receivervisor,Receivervisor会将ReceiveBlock的元数据上报ReceiverTracker,这样数据被采集存入ReceiverBlock,并且元数据也最终汇聚在ReceiverTracker(或者说汇聚到Driver端)  
* Receiver与Task  
在Spark1.5之后,就开始为一个Receiver分配一个独立的Task了.即一个Task中只会运行这个Receiver线程,这是为Receiver的高可用做准备了.因为Spark调度的任务执行单元是Task,如果某一个Receiver挂掉了,就等于这个Task挂掉了,反过来也一样.这样高可用处理就比较简单了,某个Task挂掉了,直接从当前可用的Executor中找一个然后分配一个Task过去执行就行了
* 预写日志  
在流式应用中与离线不同,无法强求一个高可靠的数据源,数据的丢失往往不可恢复.所以在Receiver写入ReceiveBlock时,本身默认是写入内存或磁盘的(默认缓存界别MemoryAndDisk).相比RDD的默认Memory级别已经安全的多了,但缓存本身无法Executor崩溃的问题,因为在Spark的BlockManager存储体系中Block无论是内存还是磁盘都是与Executor息息相关,所以引入了预写日志  
预写日志的本质是,就是在写入ReceiveBlock的同时写入一个高可用的文件系统,好处是ReceiverBlock的可以从这个高可用的文件系统恢复,但劣势是数据会写双份(如果加上源数据就三份了),并且写入文件系统的效率无论如何也不会太高  
所以用预写日志做高可用只应该是一个万般无奈的选择,最优的选择是想办法为Streaming提供一个流式的高可用数据源来代替,比如说Kafka

**计算体系**  
Streaming的计算体系由JobGenerator构成  
Streaming中DStream的Action,会将这个DStream转换为一个OutputDStream,并将其注册到DSreamGraph的outputStream(ArrayBuffer[DStream]).这一点与Core不同,在Core中Action代表触发产生一个Job,然后就开始分解出Stage然后产生Task开始执行了,在Streaming中Action仅仅只是注册执行,也就是此时不会执行而是在将来某时执行  
何时执行就是由JobGenerator控制了.JobGenerator内部维护一个定时器Timer执行,执行的间隔
就是StreamingContext中设置的时间间隔.每当JobGenerator触发一次执行,就是对DStreamGraph所有的outputStream执行generateJob,而每一个outputDStream的generateJob等于对其RDD执行Action的传名描述包装成作业Job.这样的最终结果就是作业序列Seq[Job],最后将时间维度Time,数据块(inputDStream中的ReceiveBlock),作业序列Seq[Job]共同组成JobSet,之后就交给Core的计算引擎来执行了  

## Spark性能调优  
https://blog.csdn.net/vinfly_li/article/details/79415342
**分配合适的资源**  
为Spark分配合适的资源,Core(Executor*Core)和Memory,是Spark调性能的重中之重,是一切优化的前提, Spark作为一个分布式计算框架,计算并行度一定是非常重要的,而Core就是决定Spark作业并行度的根本,而且Spark与MR不同,从某种意义上,算是架构在内存上的计算框架,内存容量对于Spark也是至关重要的  
分配出合适资源,主要从两个角度分析  
* 从集群能够给与的资源上分析  
* 从自己任务执行需要的并行度分析  

我们的集群是12台32G16核,配出的YARN集群资源总数为28G*10=280G,14*10=140核.(留2核4G作为操作系统基础运行) ,实时作业占比60%,集群总资源大约是196G98核.理论上可以配出98个2G1核,但实际我们实时作业配的是9个4核20G.  
首先是我们集群比较奇葩,16核的核足够多,32G的内存却偏少(当然这个我们没法选择).所以在YARN配置上就没有倍化核数.36核是因为我们的实时作业是对的kafka的Direct模式,kafka这边我们就配的30个分区,所以就配了36核,配出七八十核没有意义,核的意义在同时执行的最大Task数量,用不上的占有就是集群资源的浪费.而配成9个4核则是因为我们集群的奇葩特点.9个4核的9是为了凑出内存.因为YARN节点一台上限就是28G,为了留出一点空余防止随便一个离线占住就分配不出了,所以一个节点最多就能分出20G,为了尽可能占满集群允许占用的196G,所以设定了9个Executor,每个20G,再倒推核数,一个Executor最终就决定分配为9个4核20G  
总之原则就是首先分析集群给出的资源总量,并想办法尽可能的占满.无论如何,得到更多的资源总不是坏事,内存可以尽量占满,有更多的内存就能做更多的事情.而比如核,需要从自己作业的实际执行分析,如果作业没有这么大的并行度,白占核没有意义,纯粹的损人不利己  

**RDD缓存**  
有了充足的内存资源,就可以放开手脚做很多事情了.  

比如广播变量,缓存,BroadCast-join等等

对于计算重复的RDD,尽可能缓存.因为RDD的计算是溯源到数据源的,比如读取HDFS,如果不缓存每一次的重复的计算都是从连接HDFS然后拉取数据开始.

缓存时如果内存紧张可以考虑Kryo序列化缓存.序列化缓存的优势在于减少字节大小,但是会耗用更多的计算.一个简单的原则是如果内存够用就别使用序列化,我们就没有用.序列化缓存不是首选,普通缓存才是,序列化缓存是只在内存紧张的优化方案.  

还可以根据实时还是离线调一下 spark.memory.storageFaction  
总的原则是实时的存储内存比率调高一点.离线可以稍微调一点.因为实时是ReceiverBlock,在Receiver收到数据会先写入ReceiverBlock再上报Receivervisor,也就是所谓的Streaming自带缓存,而了尽可能提升实时处理速度防止积压,最好是走内存所以可以把实时的存储比率调高.而离线部分可以大致估算下广播+shuffle+缓存的总和大小调整比率适当下调,一般来说0.6有点用不了那么多 
当然不调也不一定有问题.Spark的统一内存是可以互相借用的,但预先调好阈值,可以避免一些内存借用展开的判断.最怕的是存储内存调的太高,会导致执行时频繁触发GC.因为BlockManager的MemoryStore,是一个小型的内存缓存.LRU算法,如果不占满,不主动清除是会一直在内存中不会释放的.  

**减少数据本地化等待**  
`spark.locality.wait`,默认3秒  
Spark对Task分配Executor时,会以数据本地化,也就是尽可能靠近数据方式进行.按照同Executor,同Node,Any不断尝试.但是一个比较坑的地方是,Spark的本地化机制是串行的.它是对TaskSet中的Task一个一个的遍历,每一个Task遍历又按照Block的所在地对Executor逐一尝试,这种尝试的方式还是自旋递归尝试直到超时,每一次的尝试是3秒.这就比较坑了,在我的理解中,数据本地化是一种期望而不是事实,这一点可能在流式还能做到,离线的比如HDFS,很难做到DN上刚好就有空闲的资源又这么恰好被选为你的作业的executor.理论上最坏的情况下5个级别就是15秒,100个Task差不多就是半小时,这是不能接受的   


**设置一个好的默认并行度**  
设置默认并行度为`spark.default.parallelism`,默认200.这也算个比较经典优化点了.这个配置是用来设置一切RDD中没有默认分区度的.有分区度的,它管不着.比如读HDFS,默认分区数是块数,比如KafkaDirect,RDD分区对标kafka分区,它也管不着.说的直白一点,它管的是通过par方式创建RDD集合的情况,在我们实时计算中,监控规则是定时从Redis读取并广播出去,数据量不大也就几十条,如果不设置这个,RDD分区就是200,这里设置的是9.设置成9,按个人理解我想设1的,但可惜没有意义(扯一段RDD分区计算的源码),因为Executor数量是9,所以根本设不出小于9的  

GroupByKey 和 ReduceByKey 将继承父RDD的并行度设置  

**JVM**  
Spark的每一个Executor都是一个JVM,这里也是优化的一个重点  
GC停顿主要调优指标,非常依赖GC日志(长停顿是非常危险的,无论是kafka还是shuffle,各种超时)  
剩下的就是具体问题具体分析了
* GC与堆代原理  
* 串行GC调优  
* CMC GC 调优 
* G1 GC 调优 
综合分析.我一般是首先看GC耗时理不理想,如果不理想就根据数据量和自己的代码根据经验能不能定位修复问题,或者是直接看JVM的GC日志  
按个人经验来说,一般就是调整堆大小,年轻代老年代这些  

如果是大量的MinorGC,可以适当的调高年轻代大小.一般来说,都可以适当调高年轻代大小,因为map,flatMap等等Spark的常用算子,都会产生大量的瞬时对象,年轻代太小很可能一次Map都走不完产生几次MinorGC,甚至可能会误把瞬时对象升到老年代去,导致老年代很快被占满又被迫FullGC,造成STW.STW很容易引发各种莫名其妙的问题,比如Shuffle File Not Found,比如kafka会话超时,非常麻烦  

但是MinorGC也不适应设的过大.以我们的10G为例  

`spark.core.connection.ack.wait.timeout` 默认60秒  

**使用很好的序列化方式**  
比如Kyro.  
序列化的好处有两个  减少内存或磁盘占用   和 减少网络传输  
序列化的坏处是会耗用更多的CPU来进行序列化和反序列化  
我们是使用的了序列化的,因为我们的集群较小,最大的看重不是为了占用而是为了节约网络,而CPU方面我们的集群任务不多,所以CPU方面压力比较轻  

**尽可能去shuffler**  
1. 广播变量  去shuffle  
2. join协同分区,可以减少shuffle量  



