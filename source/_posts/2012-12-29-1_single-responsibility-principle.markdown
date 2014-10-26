---
layout: post
title: "[六大设计原则] 1. Single Responsibility Principle"
date: 2012-12-29 09:14:07 +0800
comments: true
categories: [Design Patterns]
tags: [Design Pattern, SOLID, Java] 
keywords: Design Pattern, 设计模式, SRP, 单一职责原则, Single Responsibility Principle, 设计原则
description: There should never be more than one reason for a class to change.  应该有且仅有一个原因引起类的变更。（如果类需要变更，那么只可能仅由某一个原因引起） 

---
##定义

SRP，Single Responsibility Principle：  

- There should never be more than one reason for a class to change.  
>应该有且仅有一个原因引起类的变更。（如果类需要变更，那么只可能仅由某一个原因引起）

##问题由来
类T负责两个不同的职责：职责P1，职责P2。当由于职责P1需求发生改变而需要修改类T时，有可能会导致原本运行正常的职责P2功能发生故障。  

##解决方案
遵循单一职责原则。分别建立两个类T1、T2，使T1完成职责P1功能，T2完成职责P2功能。这样，当修改类T1时，不会使职责P2发生故障风险；同理，当修改T2时，也不会使职责P1发生故障风险。     
<!--more-->

##示例

- 如果一个接口包含了两个职责，并且这两个职责的变化不互相影响，那么就可以考虑拆分成两个接口。
- 方法的职责应清晰、单一。一个method尽可能制作一件事情。changeUserInfo()可拆分为changeUserName()、changeUserAddr()....

说到单一职责原则，很多人都会不屑一顾。因为它太简单了。稍有经验的程序员即使从来没有读过设计模式、从来没有听说过单一职责原则，在设计软件时也会自觉的遵守这一重要原则，因为这是常识。在软件编程中，谁也不希望因为修改了一个功能导致其他的功能发生故障。而避免出现这一问题的方法便是遵循单一职责原则。虽然单一职责原则如此简单，并且被认为是常识，但是即便是经验丰富的程序员写出的程序，也会有违背这一原则的代码存在。    
为什么会出现这种现象呢？因为有**职责扩散**。所谓职责扩散，就是因为某种原因，职责P被分化为粒度更细的职责P1和P2。此时，按照SRP 应该再新建一个类负责职责P2，但是这样会修改花销很大！除了改接口 还需要改客户端代码！所以一般就直接在原有类方法中增加判断 支持职责P2；或者在原有类中新增一个方法来处理职责P2（做到了方法级别的SRP），

例如原有一个接口，模拟动物呼吸的场景：  
``` java
class Animal {    
    public void breathe (String animal) {    
        System.out.println(animal+"呼吸空气");    
    }    
}    
public class Client {    
    public static void main (String[] args) {    
        Animal animal = new Animal();    
        animal.breathe("牛");    
        animal.breathe("羊");    
        animal.breathe("猪");    
    }    
}
```

程序上线后，发现问题了，并不是所有的动物都呼吸空气的，比如鱼就是呼吸水的。

**修改一**：修改时如果遵循单一职责原则，需要将Animal类细分为陆生动物类Terrestrial，水生动物Aquatic，代码如下：  
``` java
class Terrestrial {    
    public void breathe(String animal) {    
        System.out.println(animal+"呼吸空气");    
    }    
}    
class Aquatic {    
    public void breathe(String animal) {    
        System.out.println(animal+"呼吸水");    
    }    
}    
    


public class Client {    
    public static void main(String[] args) {    
        Terrestrial terrestrial = new Terrestrial();    
        terrestrial.breathe("牛");    
        terrestrial.breathe("羊");    
        terrestrial.breathe("猪");          
        Aquatic aquatic = new Aquatic();    
        aquatic.breathe("鱼");    
    }    
}   
```   

BUT，这样修改花销是很大的，除了将原来的类分解之外，还需要修改客户端。

**修改二**：直接修改类Animal；虽然违背了单一职责原则，但花销却小的多  
``` java
class Animal {    
    public void breathe(String animal) {    
        if ("鱼".equals(animal)) {    
            System.out.println(animal+"呼吸水");    
        } else {    
            System.out.println(animal+"呼吸空气");    
        }    
    }    
}    
    
public class Client {    
    public static void main(String[] args) {    
        Animal animal = new Animal();    
        animal.breathe("牛");    
        animal.breathe("羊");    
        animal.breathe("猪");    
        animal.breathe("鱼");    
    }    
}  
```  

这种修改方式要简单的多。但是却存在着隐患：有一天需要将鱼分为呼吸淡水的鱼和呼吸海水的鱼，则又需要修改Animal类的breathe方法，而对原有代码的修改会对调用“猪”“牛”“羊”等相关功能带来风险，也许某一天你会发现程序运行的结果变为“牛呼吸水”了。

这种修改方式直接在代码级别上违背了单一职责原则，虽然修改起来最简单，但隐患却是最大的。

**修改三**：  
``` java
class Animal {    
    public void breathe(String animal) {    
        System.out.println(animal+"呼吸空气");    
    }       
    public void breathe2(String animal) {    
        System.out.println(animal+"呼吸水");    
    }    
}    
    
public class Client {    
    public static void main(String[] args) {    
        Animal animal = new Animal();    
        animal.breathe("牛");    
        animal.breathe("羊");    
        animal.breathe("猪");    
        animal.breathe2("鱼");    
    }    
} 
```   

这种修改方式没有改动原来的方法，而是在类中新加了一个方法；虽然也违背了单一职责原则，但在方法级别上却是符合单一职责原则的，因为它并没有动原来方法的代码。

##好处

一个接口的修改只对相应的实现类有影响，对其他接口无影响；有利于系统的可扩展性、可维护性。

##问题

“职责”的粒度不好确定！  
过分细分的职责也会人为地增加系统复杂性。

##建议

对于单一职责原则，建议 接口一定要做到单一职责，类的设计尽量做到只有一个原因引起变化。  
只有逻辑足够简单，才可以在代码级别上违反单一职责原则；只有类中方法数量足够少，才可以在方法级别上违反单一职责原则；    




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