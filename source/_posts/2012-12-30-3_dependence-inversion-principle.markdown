---
layout: post
title: "[六大设计原则] 3. Dependence Inversion Principle"
date: 2012-12-30 19:41:26 +0800
comments: true
categories: [Design Patterns] 
tags: [Design Pattern, SOLID, Java] 
keywords: Design Pattern, 设计模式, 设计原则, DIP, 依赖倒置原则, Dependence Inversion Principle
description: 

---
##定义

DIP，Dependence Inversion Principle:  

- High level modules should not depend upon low level modules. Both should depend upon abstractions.  
- Abstractions should not depend upon details. Details should depend upon abstractions.

即“面向接口编程”：  

- 高层模块不应该依赖低层模块，两者都应该依赖其抽象；
>模块间的依赖通过抽象发生。实现类之间不发生直接的依赖关系（eg. 类B被用作类A的方法中的参数），其依赖关系是通过接口或抽象类产生的；
- 抽象不应该依赖细节；
>接口或抽象类不依赖于实现类
- 细节应该依赖抽象；
>实现类依赖接口或抽象类

		**何为“倒置”？**  
		依赖正置：类间的依赖是实实在在的实现类间的依赖，即面向实现编程，这是正常人的思维方式；	  
		而依赖倒置是对现实世界进行抽象，产生了抽象间的依赖，代替了人们传统思维中的事物间的依赖。

<!--more-->
##问题由来  
类A直接依赖类B，假如要将类A改为依赖类C，则必须通过修改类A的代码来达成。  
这种场景下，类A一般是高层模块，负责复杂的业务逻辑；类B和类C是低层模块，负责基本的原子操作；假如修改类A，会给程序带来不必要的风险。  

**示例（类间的耦合性）：**

例如有一个Driver，可以驾驶Benz：
```
public class Driver {  
    public void drive(Benz benz) {  
        benz.run();  
    }  
}  

public class Benz {  
    public void run() {  
        System.out.println("Benz开动...");  
    }  
} 
``` 

问题来了：现在有变更，Driver不仅要驾驶Benz，还需要驾驶BMW，怎么办？  
Driver和Benz是紧耦合的，导致可维护性大大降低、稳定性大大降低（增加一个车就需要修改Driver，Driver是不稳定的）。  

**示例（并行开发风险性）：**

如上例，Benz类没开发完成前，Driver是不能编译的！不能并行开发！

##解决办法

将类A修改为依赖接口I，类B和类C各自实现接口I，类A通过接口I间接与类B或者类C发生联系，则会大大降低修改类A的几率。  
  
上例中，新增一个抽象ICar接口，ICar不依赖于BMW和Benz两个实现类（抽象不依赖于细节）。  
1）Driver和ICar实现类松耦合  
2）接口定下来，Driver和BMW就可独立开发了，并可独立地进行单元测试


##依赖的三种写法 
**1、构造函数传递依赖对象（构造函数注入）**

```
public interface IDriver {  
    public void drive();  
}  
  
public class Driver implements IDriver {  
    private ICar car;    
    public Driver(**ICar** _car) {  
        this.car = _car;  
    }   
    public void drive() {  
        this.car.run();  
    }  
}
```  

**2、setter方法传递依赖对象（setter依赖注入）**

```
public interface IDriver{  
    public void setCar(ICar car);  
    public void drive();  
}  
  
public class Driver implements IDriver {  
    private ICar car;  
    public void setCar(**ICar** car) {  
        this.car = car;  
    }  
    public void drive() {  
        this.car.run();  
    }  
}  
```

**3、接口声明依赖对象（接口注入）**


##好处

依赖倒置可以减少类间的耦合性、降低并行开发引起的风险。

##建议

DIP的核心是面向接口编程；  
DIP的本质是通过抽象（接口、抽象类）使各个类或模块的实现彼此独立，不互相影响。  

在项目中遵循以下原则：

- 每个类尽量都有接口或抽象类
- 变量的表面类型尽量使接口或抽象类
- 任何类都不应该从具体类派生*——否则就会依赖具体类。
- 尽量不要重写父类中已实现的方法——否则父类就不是一个真正适合被继承的抽象。
- 结合里氏替代原则使用   



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
