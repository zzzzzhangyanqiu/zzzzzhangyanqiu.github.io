---
title: MyBatis源码分析-简介
tags:
  - MyBatis
date: 2018-06-30
categories:
 - MyBatis
---



## 声明 ##

本系列文章是观看了徐郡明老师的《`MyBatis`技术内幕》后加上自己的理解所写，部分语句是引用书籍中内容，在这里也推荐一下此书。

## 整体架构 ##

本文使用`mybatis3.4.2`版本，并假设读者已经有了一定的`MyBatis`使用水平，如果您还未曾使用过，建议先去网上看看教程，跑跑`demo`玩玩。

学习源码，首先要对`MyBatis`有一个结构上的认识，`MyBatis`整体架构分为三层，分别是基础层、核心层和接口层。各个层次的模块如下

![](/images/oss/MyBatis%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

其中每个模块的主要功能如下图

![MyBatis](/images/oss/picgo/MyBatis.png)

在后面的文章中，我们来从下至上的依次了解每个模块的作用及其原理，一点点的揭开`MyBatis`的神秘面纱

