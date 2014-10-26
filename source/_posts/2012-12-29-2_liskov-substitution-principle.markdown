---
layout: post
title: "[六大设计原则] 2. Liskov Substitution Principle"
date: 2012-12-29 18:07:50 +0800
comments: true
categories: [Design Patterns]
tags: [Design Pattern, SOLID, Java] 
keywords: Design Pattern, 设计模式, 设计原则, LSP, 里氏替换原则, Liskov Substitution Principle
description: 所有引用基类的地方，都能透明地替换成其子类对象。只要父类能出现的地方，子类就可以出现。里氏替换原则通俗的来讲就是：子类可以扩展父类的功能，但不能改变父类原有的功能。

---
##定义

LSP，Liskov Substitution Principle：  
  
- If for each object `s` of type `S`, there is an object `t` of type `T` such that for all programs `P` defined in terms of `T`, the behavior of `P` is unchanged when `s` is substituted for `t` when `S` is a subtype of `T`.  
- Functions that use pointers or references to base classes must be able to user objects of derived classes without knowing it.
>所有引用基类的地方，都能透明地替换成其子类对象。只要父类能出现的地方，子类就可以出现。  
    
里氏替换原则通俗的来讲就是：子类可以扩展父类的功能，但不能改变父类原有的功能。

##问题由来

引入里氏替换原则能充分发挥继承的优点、减少继承的弊端。  
<!--more-->

**继承的优点：**  

- 代码共享，减少创建类的工作量；每个子类都有父类的方法和属性；  
- 提高代码重用性；  
- 子类可以形似父类，但又异于父类；  
- 提高代码可扩展性；
- 提高产品开放性。 

 

**继承的缺点：**   
 
- 继承是侵入性的：*只要继承，就必须拥有父类的属性和方法；*  
- 降低代码的灵活性：*子类必须拥有父类的属性和方法，让子类自由的世界多了些约束；*  
- 增强了耦合性：*当父类的属性和方法被修改时，必须要考虑子类的修改。* 



**示例（继承的缺点）：**  

原有类A，实现减法功能：  
```java
class A {    
    public int func1(int a, int b) {    
        return a - b;    
    }    
}    

public class Client {  
    public static void main(String[] args) {   
        A a = new A();    
        System.out.println("100-50=" + a.func1(100, 50)); 
        System.out.println("100-80=" + a.func1(100, 80)); 
    }   
}  
```    

新增需求：新增两数相加、然后再与100求和的功能，由类B来负责  
```java
class B extends A {    
    public int func1(int a, int b) {    
        return a + b;    
    }        
    public int func2(int a, int b) {    
        return func1(a, b) + 100;    
    }    
}    
    
public class Client {    
    public static void main(String[] args) {    
        B b = new B();    
        System.out.println("100-50=" + b.func1(100, 50));    
        System.out.println("100-80=" + b.func1(100, 80));    
        System.out.println("100+20+100=" + b.func2(100, 20));    
    }    
}
```     

OOPS! 原本运行正常的相减功能发生了错误。原因就是类B在给方法起名时无意中重写了父类的方法！

>有一功能P1，由类A完成。现需要将功能P1进行扩展，扩展后的功能为P，其中P由原有功能P1与新功能P2组成。新功能P由类A的子类B来完成，则子类B在完成新功能P2的同时，有可能会导致原有功能P1发生故障。

##解决方案

LSP为继承定义了一个规范，包括四层含义：  
**1. 子类必须完全实现父类的方法**  
如果子类不能完整地实现父类的方法，或者父类的某些方法在子类中已经发生畸变；则建议不要用继承，而采用依赖、聚集、组合等关系代替继承。  
例如：父类AbstractGun有shoot()方法，其子类ToyGun不能完整实现父类的方法（玩具枪不能射击，ToyGun.shoot()中没有任何处理逻辑），则应该断开继承关系，另外建一个AbstractToy父类。

**2. 子类可以有自己的个性**  
即，在子类出现的地方，父类未必就能替代。  

**3. `重载`或实现父类方法时，输入参数可以被放大（入参可以更宽松）**   
否则，用子类替换父类后，会变成执行子类重载后的方法，而该方法可能“歪曲”父类的意图，可能引起业务逻辑混乱。  

**4. `重写`或实现父类方法时，返回类型可以被缩小（返回类型更严格）**


##建议
在实际编程中，我们常常会通过重写父类的方法来完成新的功能，这样写起来虽然简单，但是整个继承体系的可复用性会比较差，特别是运用多态比较频繁时，程序运行出错的几率非常大。  
父类中凡是已经实现好的方法（相对于抽象方法而言），实际上是在设定一系列的规范和契约，虽然它不强制要求所有的子类必须遵从这些契约，但是如果子类对这些非抽象方法任意修改，就会对整个继承体系造成破坏。而里氏替换原则就是表达了这一层含义。  

>里氏替换原则通俗的来讲就是：子类可以扩展父类的功能，但不能改变父类原有的功能。  


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