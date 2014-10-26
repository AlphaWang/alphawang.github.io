---
layout: post
title: "[六大设计原则] 6. Open Closed Principle"
date: 2012-12-31 16:47:20 +0800
comments: true
categories: [Design Patterns]
tags: [Design Pattern, SOLID, Java] 
keywords: Design Pattern, 设计模式, 设计原则, OCP, 开闭原则, Open Closed Principle   
description: Software entities like classes, modules and functions should be open for extension but closed for modifications. 一个软件实体应该通过扩展来实现变化，而不是通过修改已有的代码来实现变化。

---
##定义

Open Closed Principle:  

- Software entities like classes, modules and functions should be open for extension but closed for modifications.  
> **对扩展开放，对修改关闭。**    
> 一个软件实体应该通过扩展来实现变化，而不是通过修改已有的代码来实现变化。*——but，并不意味着不做任何修改；底层模块的扩展，必然要有高层模块进行耦合*。

“变化”可分为三种类型：  

- 逻辑变化——不涉及其他模块，可修改原有类中的方法；
- 子模块变化——会对其他模块产生影响，通过扩展来完成变化；
- 可见视图变化——界面变化，有时需要通过扩展来完成变化。

##问题由来

在软件的生命周期内，因为变化、升级和维护等原因需要对软件原有代码进行修改时，可能会给旧代码中引入错误，也可能会使我们不得不对整个功能进行重构，并且需要原有代码经过重新测试。
<!--more-->

##解决方案

当软件需要变化时，尽量通过扩展软件实体的行为来实现变化，而不是通过修改已有的代码来实现变化。  
这要求：

**1、抽象约束（要实现对扩展开放，首要前提就是抽象约束）**  

通过接口或抽象类可以约束一组可能变化的行为，并能实现对扩展开放。包含三层含义：  

- 通过接口或抽象类可以约束扩展，对扩展进行边界限定，不允许出现在接口抽象类中不存在的public方法；  
- 参数类型、引用对象尽量使用接口或抽象类，而不是实现类；
- 抽象应尽量保持稳定，一旦确定即不允许修改。

**2、元数据（metadata）控制模块行为**   
元数据，即用来描述环境和数据的数据，即配置数据。例如SpingContext。     

**3、制定项目章程** 

**4、封装变化**  
封装可能发生的变化。将相同的变化封装到一个接口或抽象类中；将不同的变化封装到不同的接口或抽象类中。

##好处  

- 易于单元测试  
>如果直接修改已有代码，则需要同时修改单元测试类；而通过扩展，则只需生成一个测试类。
- 可提高复用性
- 可提高可维护性
- 面向对象开发的要求

##建议
开闭原则是最基础的原则，前5个原则都是开闭原则的具体形态。  

<!--Google Adsense-->
<p class="meta" style="text-align:center">
	<!-- 789*90 -->
	<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
	<ins class="adsbygoogle"
	     style="display:inline-block;width:789px;height:90px"
	     data-ad-client="ca-pub-6393503301700908"
	     data-ad-slot="7806666870"></ins>
	<script>
	(adsbygoogle = window.adsbygoogle || []).push({});
	</script>
</p>