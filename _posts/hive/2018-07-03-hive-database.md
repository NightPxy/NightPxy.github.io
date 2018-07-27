---
layout: post
title:  "Hive Database"
date:   2018-07-03 13:31:01 +0800
categories: hive
tag: hive
---

* content
{:toc}



# Database 相关语法  

## Create Database  
```sql
CREATE (DATABASE|SCHEMA) [IF NOT EXISTS] database_name
[COMMENT database_comment] --数据库描述
[LOCATION hdfs_path] --数据库SourceData存储路径
[WITH DBPROPERTIES (property_name=property_value,...)]  --其它数据库设置
```

## Drop Database  
```sql
DROP (DATABASE|SCHEMA) [IF EXISTS] database_name [RESTRICT|CASCADE];
-- RESTRICT   默认使用 
-- CASCADE    删除带表的数据库
```

## Alter Database
```sql
ALTER (DATABASE|SCHEMA) database_name SET DBPROPERTIES (property_name=property_value, ...)

ALTER (DATABASE|SCHEMA) database_name SET OWNER [USER|ROLE] user_or_role; 

ALTER (DATABASE|SCHEMA) database_name SET LOCATION hdfs_path;
```

# Database 的MetaStore 字符编码集  

使用MySQL作为Hive MetaStore时,如果Meta信息使用了中文,将可能因为字符编码问题而导致中文乱码  
这时不能强行将MySQL的编码集直接调整为utf-8.(MetaStore 必须保留编码集为latin1,否则会有问题)  

此时必须:  

* 传输阶段的编码集调整为utf8mb4  
这时只能将MetaStore的中文字段单独调整为 utf8mb4 具体如下:  
```
mysql://127.0.0.1:3306/homework?serverTimezone=UTC&characterEncoding=UTF-8&useSSL=false
```

* 单独修改MetaStore中需要用到中文的字段编码集  
```sql
--表字段注解和表注解
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8mb4 ;
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8mb4 ;
--分区字段注解
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8mb4 ;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8mb4 ;
--索引注解
alter table INDEX_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8mb4 ;
```