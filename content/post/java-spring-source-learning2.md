---
title: "Spring源码之路-Reader相关类分析"
date: 2023-04-18T23:19:59+08:00
draft: true

# 文章内容摘要
description: "Spring源码学习2，Reader相关类分析"
# 文章内容关键字
keywords: "Spring BeanDefinitionReader"
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

# 1.2 Reader相关类分析

![总体的类图](/post/java-spring-source-learning2/1.PNG)

## BeanDefinitionReader
> org.springframework.**beans**.factory.support *interface*

提供了一些加载BeanDefinition的方法提供实现。这个接口更多的是建议读者按照这个标准去定义一个reader，没有强制一定要实现它的方法。
提供从Resource或者特定Resource路径下来加载Bean Definition。

### AbstractBeanDefinitionReader
> org.springframework.**beans**.factory.support *abstract class*

实现了大部分方法，但留有比较核心，需要定制的方法暴露给继承他的子类来实现。如 loadBeanDefinitions(Resource resource)
实现类有：XmlBeanDefinitionReader GroovyBeanDefinitionReader

### XmlBeanDefinitionReader
> org.springframework.**beans**.factory.xml *class extends AbstractBeanDefinitionReader*

为使用XML描述Bean读取提供的Reader。

### GroovyBeanDefinitionReader
> org.springframework.**beans**.factory.groovy *class extends AbstractBeanDefinitionReader implements GroovyObject*

> A Groovy-based reader for Spring bean definitions: like a Groovy builder, but more of a DSL for Spring configuration.

## 从Unit Test分析Spring使用XML注入Bean的过程
*以下这段代码位于**org.springframework.core.env.EnvironmentSystemIntegrationTests**这个测试类之下。大致的内容是，加载XML_PATH这个xml,然后判断是否正确加载。*

以这个类为基础研究以下XmlBeanDefinitionReader加载Bean的过程。

org.springframework.core.env.EnvironmentSystemIntegrationTests

```java
@Test
void xmlBeanDefinitionReader_inheritsEnvironmentFromEnvironmentCapableBDR() {
   GenericApplicationContext ctx = new GenericApplicationContext();
   ctx.setEnvironment(prodEnv);
   new XmlBeanDefinitionReader(ctx).loadBeanDefinitions(XML_PATH);
   ctx.refresh();
   assertThat(ctx.containsBean(DEV_BEAN_NAME)).isFalse();
   assertThat(ctx.containsBean(PROD_BEAN_NAME)).isTrue();
}
```
重点方法是 new XmlBeanDefinitionReader(ctx).loadBeanDefinitions(XML_PATH);背后调用的是AbstractBeanDefinitionReader.java中的loadBeanDefinitions方法。最后，调用到

### loadBeanDefinitions

1.  使用ThreadLocal一个HashSet变量resourcesCurrentlyBeingLoaded来判断是否正在加载。
2.  利用入参封装一个InputSource。
3.  然后调用 doLoadBeanDefinitions(InputSource, Resource)
    1. doLoadBeanDefinitions(InputSource, Resource) 会调用 doLoadDocument(inputSource, resource) 获取一个Document对象。这个类可以理解为XML的一个document形式的封装，相当于HTML的\<root\>标签入口，它让你能够层级去访问XML的内容。
    2. doLoadDocument(inputSource, resource) 
       1. 先getValidationMode
       2. DocumentLoader.loadDocument(InputSource, EntityResolver,ErrorHandler, int validationMode, boolean namespaceAware) 
       3. 最终是通过根据入参，validationMode、 namespaceAware、 创建DocumentBuilderFactory结合entityResolver、errorHandler来创建DocumentBuilder对象来解析InputSource获得Document对象。
    3. 拿到Document对象之后registerBeanDefinitions(Document, Resource),创建一个DocReader去注册BeanDefinitions。
       1. 最终调用一个doRegisterBeanDefinitions(Element root)来注册Bean。
       2. preProcessXml(root); **AOP相关**
       3. parseBeanDefinitions(root, this.delegate); 真正一层一层找Bean，DefaultBeanDefinitionDocumentReader.parseDefaultElement(Element, BeanDefinitionParserDelegate)是真正分发去看不同类型的声明。查看它究竟是import，alias，bean还是beans节点。这里很巧妙地用来递归实现。
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
      importBeanDefinitionResource(ele);
      //loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources)
   }
   else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
      processAliasRegistration(ele);
      //getReaderContext().getRegistry().registerAlias(name, alias);
   }
   else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
      processBeanDefinition(ele, delegate);
      //最终会调用到registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
   }
   else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
      // recurse
      doRegisterBeanDefinitions(ele);
      //doRegisterBeanDefinitions(ele);递归调用回去
   }
}
```
       4.postProcessXml(root); **AOP相关**