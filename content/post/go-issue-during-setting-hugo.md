---
title: "Go Issue During Setting Hugo"
date: 2023-02-05T21:35:59+08:00
draft: true

# 文章内容摘要
description: "搭建Hugo环境时，关于Go的小issue"
# 文章内容关键字
keywords: "Hugo-Golang"
# 发表日期
date: 2023-02-05
# 最后修改日期
lastmod: 2023-02-05
# 分类
categories:
 - Go
# 标签
tags:
  - Go
  - Hugo
  - bug

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
weight: 3
# 开启数学公式渲染，可选值： mathjax, katex
math: mathjax
# 开启各种图渲染，如流程图、时序图、类图等
#mermaid: true
---

## Connection fail

由于代理网站为国外网站，所以需要用以下命令来更改代理网站
> go env -w GOPROXY=https://goproxy.cn,direct

## Go install 无反应，安装的程序不能执行
由于go path未未设置在系统环境变量中，导致即便安装了程序在go_path里，执行时却无法发现。
> go env #获取GOPATH路径，将其设置为系统变量