---
layout: post
title:  "Hive Functions"
date:   2018-07-05 13:31:01 +0800
categories: hive
tag: hive
---

* content
{:toc}



# 函数的分类  

* **UDF**   
单进单出函数  
* **UDAF**   
多进单出函数 (比如Count,Sum...多个记录输出一个结果)
* **UDTF**  
单进多出函数 (比如explode,一个记录输出多行结果)

# 内置函数

## 操作符  

| 使用 | 描述 |
| ------ | ------ |
| A = B(等价A == B) | 值相等 |
| A <==> B | Equals(两个同时为Null为真,只有一个Null为假,否则Equals) |
| A <>B(等价A!=B) | 不等于(同Null为假,只有一个NUll为真,否则Equals取反) |
| A [NOT] BETWEEN B AND C | [B,C]A是否在B,C之间(包含边界B,C) |
| A IS [NOT]  NULL | A是否为Null |
| A [NOT] LIKE B | 同MySQL Like操作符 |
| A RLIKE B (等价A REGEXP B)  | 模糊匹配(正则) |
| A DIV B | A除以B的整数部分(小数直接摄取) |

## 复杂对象  

| 使用 | 描述 |
| ------ | ------ |
| map(key1, value1, key2, value2, ...) | 用指定的键值创建一个Map对象 |
| struct(val1, val2, val3, ...) | 用指定的值创建一个struct对象,字段名依次为col1.col2..... |
| named_struct(name1, val1, name2, val2, ...) | 用指定的字段名,字段值创建一个struct对象 |
| array(val1, val2, ...) | 用指定的值创建一个array对象 |
| create_union(tag, val1, val2, ...) | 使用标记参数指向的值创建一个联合类型 |
| A[n] | 获取数组对象A下标为n的值 |
| M[key] | 获取键值对对象M的键为key的值 |
| S.x | 获取结构体S的字段x的值 |
| size(Map<K.V>) size(Array<T>) | 返回Map或数组的元素个数 |
| map_keys(Map<K.V>) | 返回Map的键数组对象 |
| map_values(Map<K.V>) | 返回Map的值数组对象 |
| array_contains(Array<T>, value) | 返回Map是否包含指定的值 |
| sort_array(Array<T>) | 数组排序(升序) |

## UDF函数  

### 数学函数  

| 使用 | 描述 |
| ------ | ------ |
| round(DOUBLE a) | 返回a的四舍五入的值 |
| round(DOUBLE a, INT d)  | 返回a四舍五入保留d位小数的值 |
| bround(DOUBLE a [, INT d]) | 返回a的高斯四舍五入的值 |
| floor(DOUBLE a) | 返回a向上取整的值 |
| ceil(DOUBLE a), ceiling(DOUBLE a) | 返回a向下取整的值 |
| rand(), rand(INT seed) | 返回一个随机数 |
| greatest(T v1, T v2, ...) | 返回指定值中最大的值 |
| least(T v1, T v2, ...)  | 返回指定值中最小的值 |


# 自定义函数  

## 函数定义  

所有Hive的函数都是继承UDF类的Java子类.  
1. Maven导入  hive-exec hadoop-common  
2. 定义一个类继承UDF,并实现需要的evaluate方法  

自定义函数就是实现UDF类,并将jar注册到Hive的过程  

## 函数注册  

### 注册命名  
*注意函数名注册最好才采用全小写+_形式,因为函数名会全部转为小写,会失去驼峰*  

### 注册为临时函数  
临时函数随会话相关,所以创建的函数可以用在任意数据库中,但会话切换就会立即失败  

注册语法:  
```sql
add jar jar本地路径 ;
CREATE TEMPORARY FUNCTION function_name AS "class_name(类的全名:包+类名)";
```

### 注册为永久函数  
永久函数随数据库相关,作为数据库元数据的一部分,所以切换会话仍可使用,但必须显示声明所处的数据库  
永久函数的Jar文件是必须放入HDFS的  

注册语法:  
```sql
CREATE FUNCTION [db_name.]function_name AS class_name [USING JAR|FILE|ARCHIVE 'file_uri' [, JAR|FILE|ARCHIVE 'file_uri'] ];
```
## 函数的注册机制以及plugin  
Hive的内置函数本质都是自定义函数.  
无需注册直接使用的原因是内置函数在Hive启动时由FunctionRegistry类的静态函数自动注册了  
所以完全可以将自定义函数也打进Hive源码中,然后也在FunctionRegistry中自动注册(需要编译Hive源码哦)


## 函数删除  

```sql
DROP [TEMPORARY] FUNCTION [IF EXISTS] function_name;
```

