---
layout: post
title:  "SpringBoot - Spring - BeanFactory"
date:   2019-06-10 13:31:01 +0800
categories: java
tag: [java SpringBoot]
---

* content
{:toc}

##  概述  

BeanFactory 是 Spring容器关于对象管理的顶级接口  
负责某个具体Bean的创建,访问等职责  

在Spring设计中,容器本身不会直接参与Bean对象创建访问管理  
而是将这部分职责解耦分离,委托给Bean对应的BeanFactory来完成  

所以BeanFactory的核心是定义是  
* 根据BeanId或者Clazz信息 返回Bean  
* 根据BeanId或者Clazz信息 返回ObjectProvider (用于延迟加载获取Bean)     

### ObjectProvider     

ObjectProvider 是 ObjectFactory 一个增强版实现  

```java
@FunctionalInterface
public interface ObjectFactory<T> {
	T getObject() throws BeansException;
}

public interface ObjectProvider<T> extends ObjectFactory<T>, Iterable<T> {
	.....
	
	@Nullable
	T getIfAvailable() throws BeansException;
	
	@Nullable
	T getIfUnique() throws BeansException;
	
	.....
}
```



ObjectFactory是一个纯粹的创建获取对象的接口定义  
ObjectProvider的增强在于获取逻辑封装getIfAvailable,getIfUnique等  
非常类似Optional实现,还是在创建获取对象     

## 源码   

BeanDefinition 是Spring容器对Bean对象如何管理的规则定义  




