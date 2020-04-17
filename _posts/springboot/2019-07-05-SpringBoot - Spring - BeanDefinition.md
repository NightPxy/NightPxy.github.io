---
layout: post
title:  "SpringBoot - Spring - BeanDefinition"
date:   2019-06-10 13:31:01 +0800
categories: java
tag: [java SpringBoot]
---

* content
{:toc}

##  概述  

BeanDefinition 是Spring容器 对Bean对象如何管理的规则定义  

本质上是两个部分 
* BeanDefinition  
* BeanDefinitionRegistry  

## Spring的对象管理设计  

* BeanDefinition  
BeanDefinition是对Bean对象如何管理的描述定义  
* BeanDefinitionRegistry  
BeanDefinitionRegistry是容器对象管理的统一对外出口  

通过这个机制,所有需要纳入到容器管理的对象,都需要通过BeanDefinitionRegistry进行管理注册  
而注册管理时,就必须提交如何管理这个对象的BeanDefinition的描述定义  
容器对且只需对按这种标准进行提交的对象进行管理,包括Spring容器内部内置,也按照这个标准  
这也是我们使用Spring时一个非常重要的关于对象管理的干预和扩展点  

这是一个一流的框架设计  
框架本身是设定一种机制或者规范,然后对外按照这个机制规范对外提供服务.包括无论是外部扩展还是框架本身也是这个机制规范的遵循.从地位上讲,内置使用和外部扩展没有地位差别  
这将为框架带来稳定且强大的扩展能力

### BeanDefinition  

BeanDefinition 是Spring容器对Bean对象如何管理的规则定义  

### BeanDefinitionRegistry  

## 内置的管理注册源码解读  

容器内会有内置的管理注册  
比如依据XML配置或Properties配置,按照使用者的设置将一些对象纳入到容器管理  
这里Spring定义了诸如  XmlBeanDefinitionReader,PropertiesBeanDefinitionReader,GroovyBeanDefinitionReader等读取器,来完成指定的扫描读取注册工作  

这里以 XmlBeanDefinitionReader 为例,解读一下Spring自身如何完成一个整个的对象注册过程  
何时启动执行XmlBeanDefinitionReader,这里暂时略过,暂时以Spring容器会在合适的时候启动这个读取器  

XmlBeanDefinitionReader完成注解过程的核心流程如下  
* XmlBeanDefinitionReader本身启动时,会由容器指定Xml扫描读取的路径和容器注入BeanDefinitionRegistry  
* XmlBeanDefinitionReader完成Xml属性读取,将Xml的配置读取解析为统一的BeanDefinitionHolder  
* 将BeanDefinitionHolder们,通过容器注入的BeanDefinitionRegistry,  




### BeanDefinitionHolder  

DefaultBeanDefinitionDocumentReader

### BeanDefinition读取器  

* XmlBeanDefinitionReader  
* GroovyBeanDefinitionReader  
* PropertiesBeanDefinitionReader  

BeanDefinitionReader  

```java
public interface BeanDefinitionReader {

	BeanDefinitionRegistry getRegistry();

	@Nullable
	ResourceLoader getResourceLoader();

	@Nullable
	ClassLoader getBeanClassLoader();

	BeanNameGenerator getBeanNameGenerator();

	int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;

	int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;

	int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;

	int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;

}
```


