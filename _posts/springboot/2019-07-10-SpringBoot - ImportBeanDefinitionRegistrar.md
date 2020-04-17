---
layout: post
title:  "SpringBoot - ImportBeanDefinitionRegistrar"
date:   2019-06-10 13:31:01 +0800
categories: java
tag: [java SpringBoot]
---

* content
{:toc}

##  概述

ImportBeanDefinitionRegistrar  
该接口主要用来注册BeanDefinition, 所以是干预自定义注册Bean的主要手段之一  
很多第三方接口,都会通过这个接口,实现扫描指定的类然后将其注册到Spring容器中.比如Mybatis  


## 源码&原理


