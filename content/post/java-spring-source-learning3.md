---
title: "Spring源码之路-Bean 加载与创建"
date: 2023-04-19T23:13:51+08:00
draft: true

# 文章内容摘要
description: "Spring源码学习3，Bean 加载与创建"
# 文章内容关键字
keywords: "Spring AbstractBeanFactory postProcessor"
# 发表日期
date: 2023-04-19
# 最后修改日期
lastmod: 2023-04-19
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
# 1.3 Bean 加载与创建

## getBean
![BeanFactory](/post/java-spring-source-learning3/1.PNG)
getBean是BeanFactory接口中声明的方法，通过Bean名称或者类型，从容器中获取对应的Bean。

### AbstractBeanFactory的doGetBean方法分析

#### 在DefaultSingletonBeanRegistry几种集合

- singletonObjects,**Map<String,Object>**,bean名称与单例对象的ConcurrentHashMap(256)
- singletonsCurrentlyInCreation，**Set<String>**，记录着正在创建中的BeanName
- earlySingletonObjects，**Map<String,Object>**，bean名称与早期单例对象的ConcurrentHashMap(16) 1级缓存
- singletonFactories,**Map<String,ObjectFactory<?>>**，Bean名称和对应工厂的HashMap(16) 用来生成Bean
- singletonFactories,**Map<String,ObjectFactory<?>>**，Bean名称和对应工厂的HashMap(16) 用来生成Bean
- factoryBeanObjectCache，**Map<String,Object>**，FactoryBeanRegistrySupport下记录由工厂生成的单例bean。这些是由FactoryBean生成的Bean缓存。
- factoryBeanInstanceCache, **ConcurrentMap<String, BeanWrapper>**，记录未完整的Bean，以BeanWrapper封装着。
  
#### Bean的加载，doGetBean
##### 获取Bean实例
  1. 直接从singletonObject中获取单例对象，根据一些条件判断单例中的bean（从缓存中获取Bean实例）
     1. 其中先从singletonObjects获取，再到earlySingletonObjects中。若都没有就会从singletonFactories中获取ObjectFactory。某些对象初始化就会调用addSingletonFactory来将singletonFactory加到singletonFactories缓存中。如果最终走到singletonFacotry来获取Bean的话，会调用它的getObject方法，然后put到earlySingletonObjects，同时又从singletonFactories拿走。
![pic2](/post/java-spring-source-learning3/2.PNG)
     2. 如果第一次获取不到对象就往 上请求，getParentBeanFactory，让ParentBean帮忙获取。
     3. 利用RootBeanDefinition来注册一些依赖的Bean。同时也利用Scope来做判断加载Bean是否正确。利用dependentBeanMap，dependenciesForBeanMap来维护依赖bean与它依赖的map、以及bean与依赖它的map。
   
##### 从Bean实例中获取Object
无论最后是从哪里获取的BeanInstance，都逃不过一个统一高频的处理方法getObjectForBeanInstance来获取真正的Bean。其实就是为了处理FactoryBean的情况（通常Beanname以&开头）。首先，看看factoryBeanObjectCache是否由缓存相应Factory生成的Bean，如果没有就从这个FactoryBean调用getObect来获取。

##### 后置处理postProcess
getObjectFromFactoryBean中除了会获取到Object实例以外，还会对Object进行后置操作。对应衍生出三个步骤
  1. beforeSingletonCreation：检查inCreationCheckExclusions不包含，且singletonsCurrentlyInCreation没有同名的bean正在创建中。有则会抛出BeanCurrentlyInCreationException。
  2. postProcessObjectFromFactoryBean: AbstractAutowireCapableBeanFactory中的实现，通过getBeanPostProcessors()获取所有的postProcessor，然后遍历调用他们的postProcessAfterInitialization，遍历完之后返回，遍历期间方法一旦返回null就会直接返回。
  3. afterSingletonCreation:检查inCreationCheckExclusions不包含,singletonsCurrentlyInCreation能remove，不能就会抛出IllegalStateException。

## Create Bean
### doGetBean前
1. 根据className获取对应的Class，如果能获取到对应class名则将会copy一个新的RootBeanDefinition使用
2. Override 属性标记验证（prepareMethodOverride）
   1. 获取所有override的方法，判断每个方法对应在自己类，接口，父类上同一个方法名的数量，小于1就是有问题的，要抛异常。=1怎么只有一个方法，所以这里肯定不是重载，从代码里直接标记了不是overload,方便了后续需要根据参数来定义方法调用事，直接查看这个标记，如果为false就直接调用得了。
3. resolveBeforeInstantiation 解决初始化之前相关的事，BeanPostProcessor
   1. applyBeanPostProcessorsBeforeInstantiation，先执行。有机会产生一个代理对象，如果对应的class是有AOP增强的话。
   2. applyBeanPostProcessorsAfterInitialization，bean的后置操作，既然上一步都生成一个Bean实例了，就要按照正常流程把BeanProcessor要做的事也做完。

> **如果在这步就获得对象就会直接返回，因为对于做了AOP增强的Bean来说，Bean实例应该是一个代理对象。**

### doGetBean
4. 执行doCreateBean的方法，来到这一步证明这个只是一个普通的Bean了。
   1. 创建Bean实例的方法createInstance，执行后得到的是BeanWrapper。在这个方法中，会推断使用什么方式来实例化这个Bean。最后从中获取对应的Bean和Class。
![pic3](/post/java-spring-source-learning3/3.PNG)
   2. 上一步证明了它已经完成Bean的create，执行mergedDefinition的PostProcessor
   3. 解决循环依赖的问题，这里有一个变量：earlySingletonExpose = 是单例+允许循环依赖+当前Bean正在生成。
   4. 会将此时的Bean加入SingletonFacotries和registerSingletonObject中，从earlySingleton中remove。其中加入SingletonFacotries的getObjct方法是一个lamda表达式，他会检测是否实现AOP，若有则返回proxy对象
   5. populateBean，initializeBean（before，invoke，after）
      1. earlySingletonExposure 一些checking。throw BeanCurrentlyInCreationException exception
      2. Register Bean as Disposable
#### 注意
- 由于单例就会缓存在，SingletonFactories和registerSingletonObject中，所以只有Bean的作用域是Singleton时才会支持循环依赖。对于prototype作用域的bean来说，Spring是无法完成注入的。
- 对于singleton作用域的bean，可以设置setAllowCircularReference（false）来禁止使用循环依赖。