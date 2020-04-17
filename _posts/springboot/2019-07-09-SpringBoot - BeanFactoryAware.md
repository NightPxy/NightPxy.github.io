---
layout: post
title:  "SpringBoot - Bean的生命周期"
date:   2019-06-10 13:31:01 +0800
categories: java
tag: [java SpringBoot]
---

* content
{:toc}

## Bean的生命周期 概述  

一个对象,在JVM会经历实例化, 初始化,使用,销毁等一个固定过程,这称为对象的生命周期  
同样的,在Bean对象被Spring容器创建使用销毁的过程,也有一个Bean的生命周期  

Bean对象的生命周期,是由Spring容器,或者说Spring框架本身控制的.  
对使用方来说,Bean对象的生命周期是一个被动接受的过程  
但同时,Spring框架也提供了很多的接口和机制,来让使用方能对Bean生命周期做一些有限度的干预  

## Bean的生命周期过程  

### Bean的创建  

在Spring中,Bean不是由容器直接创建的,而是由容器委托给BeanFactory完成  
这么做的目的是,Bean的创建依赖每个Bean的独有设置,其过程千差万别.  
将创建的规则交给其对应的BeanFactory来持有,后续容器对Bean管理就可以只找BeanFactory就行了  

因为Spring这样的设计,所以创建Bean之前,其实是先创建其对应的BeanFactory,再由BeanFactory来继续创建Bean,BeanFactory后文详述,这里就简单视为Bean由容器直接创建  
这样Bean的创建过程大致如下  
* 对象实例化  
* Bean对象的容器上下文信息注入  
* 属性初始化和相应的初始化完结通知钩子  
* 根据该Bean的相关设置,首先实例化出Java对象  
* 根据Bean设置,容器对该对象进行属性初始化  
* 根据Bean的接口实现,容器分别注入相关信息  
实现BeanNameAware接口,容器会根据setBeanName(String beanId),将BeanId信息注入  
实现BeanFactoryAware接口,容器会根据setBeanFactory,将Bean当前的创建工厂信息注入  
实现ApplicationContextAware,容器会根据setApplicationContext,将容器上下文信息注入  
* 如果Bean实现了InitializingBean接口,容器将在初始化完成后,将调用afterPropertiesSet()  
* 如果Bean关联了BeanPostProcessor接口,将调用其postProcessBeforeInitialization(Object,String)
在初始化结束时,调用 

### Bean的销毁  

 

此时Bean的整个初始化完成,交由使用方使用,之后就是销毁环节了  
销毁环节既不是必然环节(Bean对象可以一直不销毁,比如单例),也是必然环节(容器销毁时销毁所有Bean)  
* 如果实现了Disposable接口,则销毁前会调用其destroy方法  
* 


##  BeanFactoryAware 概述

在Bean内部,如果需要自己当前所属的BeanFactory信息,就可以实现BeanFactoryAware接口  
这样在Bean的生命周期中,就可以被容器自动将BeanFactory实例注入进来  


## 源码&原理


