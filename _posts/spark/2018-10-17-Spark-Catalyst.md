---
layout: post
title:  "Spark-Catalyst"
date:   2018-10-17 13:31:01 +0800
categories: spark
tag: spark
---

* content
{:toc}


## 概述  

## DataSet&DataFrame  

## 编程模型  

编程模型分为两类 `语法树`和`规则`  
本质是上设计模式中的组合模式:对语法树遍历,适配执行某种规则  

### 语法树   

#### 语法树抽象  

TreeNode为顶层语法树抽象,三种特质  
* UnaryNode表示一元节点  
* BinaryNode表示二元节点  
* LeafNode表示叶节点  

语法树向下分为两个大体系  
* Expresstion 表达式体系 常量或Cast计算等等  
表达式体系为叶节点,即不带有向下递归操作    
* QueryPlan 执行计划体系   
执行计划有向下递归操作(先父或者先子递归)  
QueryPlan之下再分为LogicalPlan(逻辑执行计划)和SparkPlan(物理执行计划)  

#### LogicalPlan  

* 一元节点特质 比如Filter,Agg等  
* 二元节点特质,比如Join  

#### SparkPlan  

SparkPlan对应物理执行计划  
* LeafExecNode 物理执行叶节点 主要是数据源加载  
DataSourceScanExec.FileSourceScanExec 从HadoopFileRelations中Scan数据  
DataSourceScanExec.RowDataSourceScanExec 从一个Relation中Scan数据  
ExternalRDDScanExec 从一个RDD中Scan数据  
HiveTableScanExec 从一个Hive表中Scan数据  
* UnaryExecNode 物理执行一元节点  
FilterExec,CoalesceExec,SubqueryExec  
* BinaryExecNode 物理执行二元节点   
BroadcastHashJoinExec  
BroadcastNestedLoopJoinExec  
CartesianProductExec  
ShuffledHashJoinExec  
SortMergeJoinExec  

### 规则  

#### Rule  

Rule是一种作用在TreeNode之上的规则  

```scala
abstract class Rule[TreeType <: TreeNode[_]] extends Logging {
  ...
  def apply(plan: TreeType): TreeType
}
```

#### RuleExecutor  

RuleExecutor是Rule的执行器  
* 执行策略  
Once  该规则只执行一次  
FixedPoint 该规则可以执行指定次数   
* batches  Rule的集合  

RuleExecutor下分各种类型的规则  
* Analyzer 分析类规则  
* Optimized 优化类规则    

## 执行过程  

###  流程

* Parser  
SQL首先通过Parser解析成一颗`抽象语法树`  
* Analyzer 语法和元数据分析规则的扫描   
* Optimizer 优化规则扫描   
* SparkPlanner 翻译成物理执行计划  
* CodeGen 将物理执行计划生成真正的代码执行  

### Parser  

### Analyzer  

通过Parser解析的抽象语法树仅仅只有结构,能够大致明确这个SQL想干什么  
但此时这颗抽象语法树其实是不能使用的  

```
'Aggregate ['name], ['name, unresolvedalias('sum('age), None)]
+- 'UnresolvedRelation `people`
```

* 可以简单明确需要做一个聚合分组统计  
* 首先是语法分析,比如聚合分组,则要求必须查询列要么是分组列,要么是聚合列  
* 仅仅知道有一个聚合列是`sum('age)`,  sum函数不可识别  
* 仅仅知道数据源是`people`,但people是什么不可识别  

Analyzer一个非常重要的职责就是`resolved`  
将不可识别的目标识别出来,这依赖元数据库   
* 比如`sum`,从元数据库中扫描得知,这是一个UDAF函数,输出是一个Int  
* 比如`people`,从元数据库扫描知道是注册的一个表,注册的内容是一个JSON文件  

```
== Analyzed Logical Plan ==
name: string, sum(age): bigint
Aggregate [name#14], [name#14, sum(age#13L) AS sum(age)#18L]
+- Filter (age#13L > cast(10 as bigint))
   +- SubqueryAlias people
      +- Relation[age#13L,name#14] json
```


### Optimized  

#### 初始 

Analyzer分析之后的结果,是一个理论上可以识别并且翻译执行的可用语法树了  
但实际还有一个非常重要的环节是:`抽象语法树优化`  

比如这个执行计划就可以尝试谓词下推优化  

```
== Optimized Logical Plan ==
Aggregate [name#14], [name#14, sum(age#13L) AS sum(age)#18L]
+- Filter (isnotnull(age#13L) && (age#13L > 10))
   +- Relation[age#13L,name#14] json
```

执行谓词下推的原因非常简单  
如果没有谓词下推,则是读取全部(比如1000条),然后进行过滤取其中100条  
有了谓词下推,则是读取时扫描判定如果不满足则跳过,最终只会读取100条到内存  

#### 再探  

优化器是Catalyst核心中的核心  
从设计上说,优化器有基于规则优化与基于代价优化  

#### 基于规则优化  

规则优化是一种匹配规则,即假如满足XX条件,就将语法树做XX的变化  
所以规则优化的本质是树的一种等价变化  
比如谓词下推就是一种基于规则优化   

```scala
object PushPredicateThroughJoin extends Rule[LogicalPlan] with PredicateHelper {
    ...
    def apply(plan: LogicalPlan): LogicalPlan = plan transform {
    // 匹配规则 Filter(filterCondition, Join(left, right, joinType, joinCondition))
	// 这里规则其实是 join ... where ... 模式
    case f @ Filter(filterCondition, Join(left, right, joinType, joinCondition)) =>
      val (leftFilterConditions, rightFilterConditions, commonFilterCondition) =
        split(splitConjunctivePredicates(filterCondition), left, right)
      joinType match {
        case _: InnerLike =>
          ... // 内连接下推 两个数据源都可以下推
        case RightOuter =>
          ... // 右连接下推,以右端为基准.所以右端本身,左端为左端本身+右端
        case LeftOuter | LeftExistence(_) =>
          ...// 左连接下推  
        case FullOuter => f // DO Nothing for Full Outer Join
        case NaturalJoin(_) => sys.error("Untransformed NaturalJoin node")
        case UsingJoin(_, _) => sys.error("Untransformed Using join node")
      }

    //匹配规则 Join(left, right, joinType, joinCondition)
    // 这里规则其实是 join...on...模式	
    case j @ Join(left, right, joinType, joinCondition) =>
      val (leftJoinConditions, rightJoinConditions, commonJoinCondition) =
        split(joinCondition.map(splitConjunctivePredicates).getOrElse(Nil), left, right)

      joinType match {
        case _: InnerLike | LeftSemi =>
          // push down the single side only join filter for both sides sub queries
          val newLeft = leftJoinConditions.
            reduceLeftOption(And).map(Filter(_, left)).getOrElse(left)
          val newRight = rightJoinConditions.
            reduceLeftOption(And).map(Filter(_, right)).getOrElse(right)
          val newJoinCond = commonJoinCondition.reduceLeftOption(And)

          Join(newLeft, newRight, joinType, newJoinCond)
        case RightOuter =>
          ...
        case LeftOuter | LeftAnti | ExistenceJoin(_) =>
         ...
        case FullOuter => j
        case NaturalJoin(_) => sys.error("Untransformed NaturalJoin node")
        case UsingJoin(_, _) => sys.error("Untransformed Using join node")
      }
  }
}
```

#### 基于代价优化(CBO)  

基于代价优化是选择代价最小的物理执行计划  
能做到这一步的前提是,至少是要能大致预估出不同物理执行计划的代价消耗  
而这一步依赖与统计信息   
* 数据条目数,数据总大小以及推测出的单条数据大小等  
* 数据的最大值,最小值等等  
总之可以预获得的信息越多,代价推测就越精确,选择也会更加有效  

例如通过数据条目数,数据总大小,可以推测出单条数据大小  
通过最大值和最小值,可以推测出某个查询可能会查询出多少数据等等  


最常见的代价优化选择就是Join了  
* ShuffleJoin与BoardCastJoin  
Shuffle的高消耗会让我们尽可能的避免Shuffle,如果小于某个阈值,则可以优化为先广播再join.这个过程就是代价优化的一种  
* 没有CBO的情况下,只能通过表的原始大小进行判定  
但如果在CBO情况下,一个1G的数据集经过过滤后还剩10M,则依然可以进行广播  
* 优化join顺序,也就是join移位  
比如第一二两个join输出数据集很大,但第三个join之后数据集变小,则可以先进行第三个join  

* BroadcastHashJoinExec  
* BroadcastNestedLoopJoinExec  
* ShuffledHashJoinExec  
* SortMergeJoinExec  

### SparkPlanner    

经过优化后,就可以生成物理执行计划了  
物理执行计划就是逻辑执行计划真正翻译成Spark执行的计划  

```
== Physical Plan ==
*(2) HashAggregate(keys=[name#14], functions=[sum(age#13L)], output=[name#14, sum(age)#18L])
+- Exchange hashpartitioning(name#14, 5)
   +- *(1) HashAggregate(keys=[name#14], functions=[partial_sum(age#13L)], output=[name#14, sum#29L])
      +- *(1) Project [age#13L, name#14]
         +- *(1) Filter (isnotnull(age#13L) && (age#13L > 10))
            +- *(1) FileScan json [age#13L,name#14] Batched: false, Format: JSON, Location: InMemoryFileIndex[file:/D:/data/people.json], PartitionFilters: [], PushedFilters: [IsNotNull(age), GreaterThan(age,10)], ReadSchema: struct<age:bigint,name:string>
```

### 生成执行   

#### CodeGen  

CodeGen是动态字节码技术,用以在Scala这种静态语言达到动态语言执行的效果  

CodeGen是对物理执行计划的翻译执行简化,将物理计划的执行转化成字符串拼接  

以 `concat_ws(",",$"name",$"age")` 这个函数为例  
CodeGen代码如下  

```
UTF8String[] project_args_0 = new UTF8String[2];
 
boolean scan_isNull_1 = scan_row_0.isNullAt(1);
UTF8String scan_value_1 = scan_isNull_1 ? null : (scan_row_0.getUTF8String(1));
	 if (!scan_isNull_1) {
	   project_args_0[0] = scan_value_1;
	 }
   
boolean project_isNull_3 = false;
UTF8String project_value_3 = null;
if (!false) {
  project_value_3 = UTF8String.fromString(String.valueOf(scan_value_0));
}
if (!project_isNull_3) {
  project_args_0[1] = project_value_3;
}
   
UTF8String project_value_0 = UTF8String.concatWs(((UTF8String) references[2] /* literal */), project_args_0);
boolean project_isNull_0 = project_value_0 == null;
```


#### WholeCodeGen  

WholeCodeGen是CodeGen的优化方案  
CodeGen的本质是动态类生成与发射调用,但如果粒度太细(比如以方法生成一个CodeGen),会引起大量的动态类生成与反射,所以有WholeCodeGen,也就是以Stage为单位,对一个Stage产生的多个CodeGen优化为一次生成    