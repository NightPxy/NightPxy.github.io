---
layout: post
title:  "SpringBoot - Spring 概览"
date:   2019-06-10 13:31:01 +0800
categories: java
tag: [java SpringBoot]
---

* content
{:toc}

## Spring 概述  

Spring 是一个非常优秀的IOC框架 
基于IOC框架的特质,Spring本质上提供两个方面的服务  
* 对象管理以及注入依赖  
* IOC过程中的AOP功能实现  

## Spring容器  

容器的本质是Spring定义的一系列机制,比如配置扫描机制,注解扫描机制  

ApplicationContext  

## Bean对象管理  

被Spring容器管理的对象,称之为Bean对象  
站在容器的角度,对Bean对象的管理无非是   
* 对象管理(对象创建,对象销毁)  
* 对象关系管理  

### 对象管理  

对象管理,就是 如何管理对象的规则,以及根据规则创建目标对象和销毁  

在这方面,Spring的核心处理是  
* BeanDefinition 对象管理的规则定义  
比如Bean的目标Clazz,单例还是多例对象,是否延迟加载,前置依赖等等  
* BeanDefinitionRegister  
将BeanDefinition注册到容器的能力,也是我们常用的Spring扩展点 
通过自定义BeanDefinitionRegister将自己的BeanDefinition注册到容器,来完成自定义对象管理  
* BeanFactory  


