---
layout: post
title:  "HBase-优化-RowKey"
date:   2018-06-02 13:31:01 +0800
categories: java
tag: [java]
---

* content
{:toc}


## Region    

Region 是 HBase 分布式和负载均衡的最小单元,也就是数据获取和分布是以Region为单位的..(注意不是最小存储单元)  

Region的核心就是大小和数量问题  
 


一个理想的完整Region设计是这样的  
* 为Region设计一个合适的大小,考虑提前生成一部分预分区  
较少的,有限的分区拆分是无可避免的,但提前预分区可以极大的提高增长的前期缓慢环节  
* 保证RowKey均匀的落入所有的Region  
避免让一个Region处理过多的数据  
* 合适的过期策略.  
让数据合理的过期可以极大的减少因为累积造成的RegionSplit  


### RegionSplit  

RegionServer的拆分策略是将该Region下线,然后拆分,最后将其子Region重新加入原来RegionServer的Meta信息中,最后汇报HMaster    

Region拆分是在运行时发生的(不会出现在Major合并中),并且在Region拆分过程中,Region会短暂的对外不可用.所以Region拆分是应该尽力避免的,比较常见的解决思路是 

* 合适的Region大小 
默认首先一个Region,并依据FlushSize `min(Region上限,(Region膨胀次数^2)*FlushSize)` 膨胀   

* 预分区  

### 合适的Region大小  

默认情况下,Region首先是一个,然后不断膨胀(膨胀策略是依据FlushSize `min(hbase.hregion.max.filesize,(Region膨胀次数^2)*FlushSize)`),之后就会发送RegionSplit  

**Region过多**
* 事实上受限集群物理环境,RegionServer的数量其实是固定的,越多的Region就代表单个RegionServer需要管理的Region越多,拿取数据需要跨越的RegionServer也会越多,官方建议是一个RegionServer最好不要管理超过100个Region  
* 每一个Region都有一个MemStore  
* 从计算引擎的角度将,一个Region视同HDFS的一个Block  

**Region过少**  
* Region的数量少了就类似并发度不够的情况,发挥不了集群分布式的能力.(例如10台集群,5个Region则始终只能发挥出5台的能力)  
* Region过少也会影响也会因压力不够分散而导致更多的数据热点问题 

**Region数量参考**
计算Region数量的公式参考如下  

```
( RS堆内存 * hbase.regionserver.global.memstore.size) / (hbase.hregion.memstore.flush.size * (# column families))
```

即假设RS16GB,默认情况下推荐`16384*0.4/128m = 51 ` 个Region

hbase.hregion.max.filesize不宜过大或过小，经过实战，生产高并发运行下，最佳大小5-10GB！关闭某些重要场景的HBase表的major_compact！在非高峰期的时候再去调用major_compact，这样可以减少split的同时，显著提供集群的性能，吞吐量、非常有用

### 预分区  

因为Region的本质是RowKey的范围段,所以所谓的预分区就是提前分好一部分范围段来容纳分区  

```shell
# 该语句将创建四个分区 
# * -> 100
# 100 -> 200 
# 200 -> 300 
# 300 -> * 
create 'split01','cf1',SPLITS=>['100','200','300']
```



