---
layout: post
title: "[六大设计原则] 4. Interface Segrefation Principle"
date: 2012-12-31 16:11:26 +0800
comments: true
categories: [Design Patterns]
tags: [Design Pattern, SOLID, Java] 
keywords: Design Pattern, 设计模式, 设计原则, ISP, 接口隔离原则, Interface Segregation Principle  
description: Clients should not be forced to depend upon interfaces that they don't use. The dependency of one class to another one should depend on the smallest possible interface.  客户端只依赖于它所需要的接口；它需要什么接口就提供什么接口，把不需要的接口剔除掉。类间的依赖关系应建立在最小的接口上。即，接口尽量细化，接口中的方法尽量少 

---

##定义

Interface Segregation Principle:  

- Clients should not be forced to depend upon interfaces that they don't use.  
>客户端只依赖于它所需要的接口；它需要什么接口就提供什么接口，把不需要的接口剔除掉。  

- The dependency of one class to another one should depend on the smallest possible interface.  
>类间的依赖关系应建立在最小的接口上。  
>即，接口尽量细化，接口中的方法尽量少

##问题由来

类`A`通过接口`I`依赖类`B`，类`C`通过接口`I`依赖类`D`。如果接口`I`对于类`A`和类`B`来说不是最小接口，则类`B`和类`D`必须去实现他们不需要的方法。  
<!--more-->

##解决方案

将臃肿的接口I拆分为独立的几个接口，类A和类C分别与他们需要的接口建立依赖关系。包含4层含义：  

**1、接口要尽量小**

不能出现Fat Interface；但是要有限度，首先不能违反单一职责原则（不能一个接口对应半个职责）。  

**2、接口要高内聚**

在接口中尽量少公布public方法。  
接口是对外的承诺，承诺越少对系统的开发越有利。  

**3、定制服务**

只提供访问者需要的方法。例如，为管理员提供IComplexSearcher接口，为公网提供ISimpleSearcher接口。  

**4、接口的设计是有限度的**


##建议

- 一个接口只服务于一个子模块或业务逻辑；  
- 通过业务逻辑压缩接口中的public方法；  
- 已被污染了的接口，尽量去修改；若变更的风险较大，则采用适配器模式转化处理；  
- 拒绝盲从

##与单一职责原则的区别

二者审视角度不同：  
  
- 单一职责原则要求的是类和接口职责单一，注重的是职责，这是业务逻辑上的划分；  
- 接口隔离原则要求接口的方法尽量少。。。