---
layout: post
title:  "SpringBoot自动加载"
date:   2019-06-01 13:31:01 +0800
categories: java
tag: [java]
---

* content
{:toc}

##  


SpringBoot自动加载机制,是SpringBoot的核心机制之一  
SpringBoot自动加载,是依赖`spring-boot-autoconfigure`模块完成的  

https://blog.csdn.net/yeyinglingfeng/article/details/87790700

## 机制流程  

### 启动  

自动加载的启动,来自SpringBoot启动时的上下文刷新时  

```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

##  配置类定义  

以Redis为例,配置类如下  

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {
  ...
}
```

注意的是,配置类并不在Redis模块`spring-data-redis`,而是在模块`spring-boot-autoconfigure`中  
这代表着无论是否使用Redis,Redis配置类是始终跟随Project走的  
但是这并不意味着会真正加载Redis配置,原因在于加载条件是`@ConditionalOnClass(RedisOperations.class)`,而`RedisOperations.class`是在真正的Redis模块中的  
也就是说,这个配置会始终跟随Project,但必须在真正引入Redis模块,才会真正被加载,否则加载将被跳过  
