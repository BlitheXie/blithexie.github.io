---
title: "Spring源码之路-容器加载相关类简单介绍"
date: 2023-02-26T21:35:59+08:00
draft: true

# 文章内容摘要
description: "Spring源码学习1，容器加载相关类简单介绍"
# 文章内容关键字
keywords: "Spring BeanFactory"
# 发表日期
date: 2023-02-26
# 最后修改日期
lastmod: 2023-02-26
# 分类
categories:
 - Java
 - Spring源码分析
# 标签
tags:
  - Java
  - Spring

# 原文作者
author: Blithe Xie
# 原文链接
#link:
# 图片链接，用在open graph和twitter卡片上
#imgs:
# 在首页展开内容
expand: true
# 外部链接地址，访问时直接跳转
#extlink:
# 在当前页面关闭评论功能
#comment:
#  enable: false
# 关闭当前页面目录功能
# 注意：正常情况下文章中有H2-H4标题会自动生成目录，无需额外配置
#toc: false
# 绝对访问路径
#url: "{{ lower .Name }}.html"
# 开启文章置顶，数字越小越靠前
weight: 2
# 开启数学公式渲染，可选值： mathjax, katex
math: mathjax
# 开启各种图渲染，如流程图、时序图、类图等
#mermaid: true
---


# 1.1 容器加载相关类分析
## 总体类图

![总体的类图](/post/java-spring-source-learning1/1.png)

## BeanDefinitionRegistry
![总体的类图](/post/java-spring-source-learning1/2.PNG)

### AliasRegistry

org.springframework.**core**; \<Interface\>

管理alias的公共接口，服务于一个另一个超级接口BeanDefinitionRegistry

### BeanDefinitionRegistry

org.springframework.**beans**.factory.support;\<Interface\> extends AliasRegistry

Bean声明注册中心接口，继承了AliasRegistry接口，所以拥有alias管理功能。同时拥有bean定义相关管理的功能；如：
![BeanDefinitionRegistry方法](/post/java-spring-source-learning1/3.PNG)

### BeanDefinition

org.springframework.**beans**.factory.config;\<Interface\> extends AttributeAccessor, BeanMetadataElement

最小的Bean定义接口，用于记录描述一个bean实例的信息以及进一步实现的信息。

### AttributeAccessor 

org.springframework.**core**; \<Interface\>

属性的访问接口，拥有属性的CRUD，以及一个compute。Bean实现它相当于Bean就有了属性的相关功能。

### BeanMetadataElement
org.springframework.**beans**;\<Interface\>

元数据接口，只有getSource一个默认方法。默认实现会返回一个null。
实现这个接口相当于暴露一个方法到外面，告诉人家包装到最底层的object是什么。

## SimpleAliasRegistry
org.springframework.**core**;\<Class\>

这个类是AliasRegistry的简单实现。利用ConcurrentHashMap来存储alias的mapping，key-alias,value-name。斯所以一个name有很多alias。

```
/** Map from alias to canonical name. */
private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);

checkForAliasCircle(name, alias);

```
在 registerAlias 中有一个 checkForAliasCircle(name,alias)来防止互为别名。
allowAliasOverriding() 获取是否可以覆盖alias，这里简单设置为ture。提醒，这里alias是有机会拥有自己的alias的。

## DefaultSingletonBeanRegistry
![DefaultSingletonBeanRegistry相关类图](/post/java-spring-source-learning1/4.png)

### SingletonBeanRegistry 
org.springframework.beans.factory.config;\<Interface\>

管理Singleton的基础接口，暴露singleton CRUD的操作。

![SingletonBeanRegistry](/post/java-spring-source-learning1/5.png)

### DefaultSingletonBeanRegistry

org.springframework.**beans**.factory.support;\<Class\> extends **SimpleAliasRegistry** implements **SingletonBeanRegistry**

主要维护以下几个concurrentHashMap：singletonObjects、singletonFactories、earlySingletonObjects、以及三个set: registeredSingletons、singletonsCurrentlyInCreation、inCreationCheckExclusions
以完成加减SingletonBean的完整的基础流程。其中包含依赖bean的解析过程。
该bean主要是提供一个默认实现。

### FactoryBeanRegistrySupport
org.springframework.**beans**.factory.support；abstract \<Class\> extends **DefaultSingletonBeanRegistry**

维护factoryBeanObjectCache用于存储factoryBean，它用于支持需要管理FactoryBean的注册中心。同时又拥有DefaultSingletonBeanRegistry的能力。

## BeanFactory
org.springframework.**beans**.factory\<Interface\> 

这个是全部Spring容器的根接口，包括之后的configurableBeanFactory和ListableBeanFactory。
这个接口的说明信息上还包含了bean生命周期中14个启动过程和3个消亡过程，一个标准的BeanFactory应该尽可能的覆盖这些过程。
这个接口显然是一个工厂接口，所以基本是获取Bean的实例或属性方法。

常量：*FACTORY_BEAN_PREFIX*用来标记 是否factoryBean，值为“&”

### HierarchicalBeanFactory
org.springframework.**beans**.factory\<Interface\>  extends **BeanFactory**

就是一个普通的BeanFactory加了等级方法： getParentBeanFactory。用来划分BeanFactory等级的接口。
包含一些关于自动注入的方法，生命周期方法等。\<To Be Enrich\>

### ListableBeanFactory
org.springframework.**beans**.factory\<Interface\>  extends **BeanFactory**

可以罗列所有bean信息的功能的BeanFactory接口。

### StaticListableBeanFactory
org.springframework.**beans**.factory.support;\<Class\> implements **ListableBeanFactory**

一个ListableBeanFactory的简单实现。\<To Be Enrich\>

### AutowireCapableBeanFactory
org.springframework.**beans**.factory\<Interface\>  extends **BeanFactory**

同样继承了BeanFactory，拥有自动注入相关方法的BeanFactory接口。定义了多个常量来，用于描述自动注入的方式。还有一些创建Bean相关的方法。

### ConfigurableBeanFactory
org.springframework.**beans**.factory.config; \<Interface\> extends **HierarchicalBeanFactory**, **SingletonBeanRegistry**

这个接口既有单例Bean的注册功能（我告诉你，我是单例Bean）又有层级BeanFactory（可以从中获取Bean，或者从父BeanFactory中获取）
除此之外，它自身还丰富了一些与bean生命周期相关的方法。设置Classloader,设置父BeanFactory，设置BeanPostProcessor等等。

![ConfigurableBeanFactory](/post/java-spring-source-learning1/6.png)

### AbstractBeanFactory
org.springframework.**beans**.factory.support \<abstract class\> extends **FactoryBeanRegistrySupport** implements **ConfigurableBeanFactory**
这是BeanFactory的抽象基类，实现了Configurable的所有方法\<To Be Enrich\>

## DefaultListableBeanFactory

![DefaultListableBeanFactory](/post/java-spring-source-learning1/7.png)

org.springframework.**beans**.factory.support \<Class\> extends **AbstractAutowireCapableBeanFactory**
      implements **ConfigurableListableBeanFactory**, **BeanDefinitionRegistry**, **Serializable**

*configurelistablebeanfactory和BeanDefinitionRegistry接口的Spring默认实现:
一个基于bean定义元数据的成熟的bean工厂，可通过后处理器扩展。

典型的用法是在访问bean之前先注册所有bean定义(可能是从bean定义文件中读取)。因此，在本地Bean定义表中，按名称查找Bean是一种成本低廉的操作，操作对象是预先解析的Bean定义元数据对象*。

### ConfigurableListableBeanFactory
org.springframework.**beans**.factory.config \<Interface\> extends **ListableBeanFactory**, **AutowireCapableBeanFactory**, **ConfigurableBeanFactory**

### AbstractAutowireCapableBeanFactory
org.springframework.**beans**.factory.support;\<abstract class\> extends **AbstractBeanFactory**
      implements **AutowireCapableBeanFactory** 

*提供bean创建(带有构造函数解析)、属性填充、连接(包括自动装配)和初始化。处理运行时bean引用，解析托管集合，调用初始化方法，等等。支持根据名称自动装配构造函数、属性和类型自动装配属性。*
