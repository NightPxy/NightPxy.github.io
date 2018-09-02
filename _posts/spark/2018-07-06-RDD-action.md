---
layout: post
title:  "RDD-Action"
date:   2018-07-06 13:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}


## 概述  

在Spark中,所有的 transformation(转换) 都是Lazy执行的 (*详见上章*)  
必须触发一个action(动作),才会产生一个执行作业(job)来真正执行所有的操作  

## 常用action  

### reduce  
RDD中元素前两个传给输入函数，产生一个新的return值，新产生的return值与RDD中下一个元素组成两个元素，再被传给输入函数，直到最后只有一个值为止  

```scala
val rdd = sc.parallelize(1 to 100,2);
val value = rdd.reduce((v1,v2)=>v1+v2)
println(value)
//输出结果 5050
```
**reduce 与 reduceByKey**
>  严格说,这两者没有可比性.虽然这两者很相似 但有本质的区别   
>  reduceByKey是一个*transformation(转换)*操作,返回的是RDD  
>  reduce是一个action(动作)操作,返回的是数据结果  


### collect  
将一个RDD的所有元素以数组的形式发回driver端  

**注意**
>  这个RDD必须是足够小的数据集,否则很容易将driver端的内存撑爆.  
>  一般建议用take  

### count
返回一个RDD的元素的个数  

### first  
返回一个RDD的第一个元素  

### take(n)
返回一个RDD的前N个元素

### takeSample  
返回一个RDD的百分比抽样  
语法: ```takeSample(withReplacement, num, [seed])```  
withReplacement 标识元素是否抽样后,是否再放回RDD以供多次使用  

### takeOrdered  
返回一个RDD按照设定的排序规则后的前N个元素  
语法: ``` takeOrdered(n, [ordering])```  

### countByKey  
返回一个RDD的按照Key分组后的每组计数  
只支持键值对类型  

### foreach  
在数据集的每个元素上运行函数func 通常用于处理副作用，如更新累加器或与外部存储系统交互  

**注意**  
> 不可以修改foreach()之外的累加器之外的变量 (见RDD-变量一节)  

### saveAsTextFile(path)  
将一个RDD的全部元素文本方式写入本地文件,HDFS或其它任何Hadoop支持的存储系统中.  
(每行等于每个元素调用toString()的结果)  

### saveAsSequenceFile(path)  
将一个RDD的全部元素二进制方式写入本地文件,HDFS或其它任何Hadoop支持的存储系统中.  
在Scala中，它还可以用于隐式转换为可写类型的类型(Spark包含对基本类型的转换，如Int、Double、String等)  

### saveAsObjectFile(path)  
将一个RDD的全部元素Java序列化以简单的格式写入本地文件,HDFS或其它任何Hadoop支持的存储系统中  
可以使用SparkContext.objectFile()加载这些元素  