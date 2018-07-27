---
layout: post
title:  "Hive Table"
date:   2018-07-04 13:31:01 +0800
categories: hive
tag: hive
---

* content
{:toc}



# 表的基本操作  

## 默认存储位置  

用户可以自定义表存储SourceData的HDFS位置  
如果不指定,将默认存储于 ```/usr/hive/warehouse/数据库名.db/表名下```

## 内/外部表  
Hive依据本身对表的托管力度,将表分为两大类 **内部表** 和 **外部表**  

**内部表(MANAGED_TABLE)**  
> Hive同时管理MetaData和SourceData.表删除时,Hive将同时删除MetaData和SourceData  

**外部表(EXTERNAL)**  
> Hive只管理MetaData.表删除时,Hive只删除MetaData.不会影响到SourceData  

## 表创建  

### 语法建表  

```sql
-- 表定义: TEMPORARY: 临时表 EXTERNAL:外部表
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name    
  [(col_name data_type [COMMENT col_comment], ... [constraint_specification])]
[COMMENT table_comment]  --表描述
[PARTITIONED BY (col_name data_type [COMMENT col_comment],...)] --表分区设置
[LOCATION hdfs_path] --设定数据文件存于HDFS上的位置
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY '\001'  --列分隔符; 
```



### 查询结果建表  
依据查询的结果创建表,并将结果数据写入表中  
> 查询建表会执行MapReduce  
> 目标表必须不存在
```sql
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\001' 
[LOCATION hdfs_path]
AS
SELECT * FROM xxx
```



### 拷贝表结构(like)  

会创建结构完全相同的表，但是没有数据
> like拷贝表结构不用执行MapReduce  

```sql
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
LIKE existing_table_or_view_name
[LOCATION hdfs_path];
```

## 表数据导入  

> 表数据导入的前提是表已经存在  

### LOAD DATA  
从一个文件中导入数据到目标表  
```sql
LOAD DATA [LOCAL] INPATH '源文件路径' [OVERWRITE] INTO TABLE 目标表  
```
[LOCAL]
> 加上 LOCAL 表示从本地文件系统上导入(拷贝拷入)  
> 不加  LOCAL 表示从HDFS导入 (**移动拷入,源文件会被删除**)  

[OVERWRITE]  
> 加上 OVERWRITE 表示数据全覆盖  
> 不加 OVERWRITE 表示数据以追加形式写入  

### INSERT INTO TABLE    
从一个已存在的表中导入数据到目标表  
```sql
insert into table 目标表[分区设置]
select 字段... from 源表 ;
```

## 表分区  

### 概述  
分区在HDFS中体现为表的子文件夹  
分区的最大优势在于降低IO,以分区字段筛选数据时,可以直接跳过读取和计算整个分区下的数据  
特别是大数据情况下,良好的分区可以显著的提供计算性能  

### 分区表语法  
```sql
create table order_partition
.....常规建表
partitioned by (分区字段,分区字段类型........) --分区字段设置
......
--语法要求:分区字段不得与任何非分区字段同名
```

### 静态分区  
导入数据时,手动指定分区,将数据直接放入目标分区中  

```sql
LOAD DATA LOCAL INPATH '/xxpath/xx.txt' 
OVERWRITE INTO TABLE order_partition 
PARTITION(event_month='2014-05'); --手动设置分区字段和分区键
```

### 动态分区  
导入数据,根据指定的分区字段自动的进去不同的目标分区  

Hive默认不支持动态分区,使用动态分区时必须先打开限制,如下:  
```sql
--set hive.exec.dynamic.partition; 查看是否打开了动态分区： 
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict; 
SET hive.exec.max.dynamic.partitions=100000;--如果自动分区数大于这个参数，将会报错)
SET hive.exec.max.dynamic.partitions.pernode=100000;
```
导入数据时指定分区字段即可:  
```sql
insert into table 目标分区表 partition(deptno) --指定分区字段
--动态分区明确要求：分区字段(deptno)写在select的最后面
select xxx,deptno from 源数据表 ;

--LOAD DATA 同上
```

### 刷新分区  
Hive的分区信息是根据 MetaStore 而来,所以在某些数据已经进入之后的场景,也无法体现分区  
这时需要手动刷新增加分区信息

#### 刷新表全分区  

```sql
MSCK REPAIR TABLE table_name
--Hive会检测如果HDFS目录下存在但表的metastore中不存在的partition元信息，更新到metastore中
```

#### 刷新表指定分区  

```sql
--一次一个分区
ALTER TABLE table_name ADD IF NOT EXISTS PARTITION(dt='2018-01-01') LOCATION 'xxpath';

--一次多个分区
ALTER TABLE table_name ADD 
PARTITION (dt='2018-01-01', country='cn') location '/cn/part180101' 
PARTITION (dt='2018-01-02', country='cn') location '/cn/part180102' ;
```

#### 删除分区  

```sql
ALTER TABLE table_name DROP IF EXISTS PARTITION (dt='2018-01-01');
```