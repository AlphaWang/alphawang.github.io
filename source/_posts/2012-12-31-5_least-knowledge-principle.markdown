---
layout: post
title: "[六大设计原则] 5. Least Knowledge Principle"
date: 2012-12-31 16:31:41 +0800
comments: true
categories: [Design Patterns]
tags: [Design Pattern, SOLID, Java] 
keywords: Design Pattern, 设计模式, 设计原则, LKP, 最少知识原则, Least Knowledge Principle, Law of Demeter  
description: 最少知识原则（Least Knowledge Principle），又称迪米特法则（LoD，Law of Demeter），是指一个对象应该对其他对象有最少的了解。一个类对自己依赖的类知道的越少越好。 也就是说，对于被依赖的类来说，无论逻辑多么复杂，都尽量地的将逻辑封装在类的内部，对外除了提供的public方法，不对外泄漏任何信息。

---
##定义

最少知识原则（Least Knowledge Principle），又称迪米特法则（LoD，Law of Demeter），是指一个对象应该对其他对象有最少的了解。
一个类对自己依赖的类知道的越少越好。  
也就是说，对于被依赖的类来说，无论逻辑多么复杂，都尽量地的将逻辑封装在类的内部，对外除了提供的public方法，不对外泄漏任何信息。

##问题由来

类与类之间的关系越密切，耦合度越大，当一个类发生改变时，对另一个类的影响也越大。
<!--more-->

##解决方案

迪米特法则包含4层含义：  

**1、只和朋友交流**  

Only talk to your immediate friends. 两个对象之间的耦合就成为朋友关系。即，出现在成员变量、方法输入输出参数中的类就是朋友；局部变量不属于朋友。  

>不与无关的对象发生耦合！  

方针：不要调用从另一个方法中返回的对象中的方法！只应该调用以下方法：
  
- 该对象本身的方法  
- 该对象中的任何组件的方法  
- 方法参数中传入的对象的方法  
- 方法内部实例化的对象的方法  



【例】：Teacher类可以命令TeamLeader对Students进行清点，则Teacher无需和Students耦合，只需和TeamLeader耦合即可。

【反例】：  
```java 反例
public float getTemp(){  
     Thermometer t = station.getThermometer(); //温度计对象 
     return t.getTemp();  
}
```
 
 
客户端不应该了解气象站类中的温度计对象；应在气象站类中直接加入获取温度的方法。  
改为：
```java 修改后
public float getTemp(){  
      return station.getTemp();  
}
```

**2、朋友间也应该有距离**  

即使是朋友类之间也不能无话不说，无所不知。  

>一个类公开的public属性或方法应该尽可能少！

**3、是自己的就是自己的**   

如果一个方法放在本类中也可以、放在其他类中也可以，怎么办？  

>如果一个方法放在本类中，既不增加类间关系，也对本类不产生负面影响，就放置在本类中。

**4、谨慎使用Serializable**  

否则，若后来修改了属性，序列化时会抛异常NotSerializableException。

##建议

迪米特法则的核心观念是：类间解耦。  
其结果是产生了大量中转或跳转类。