---
layout: post
title: "How String in Switch Works in Java 7"
date: 2014-12-20 10:02:33 +0800
comments: true
categories: [Java]
tags: [Java, Source Code, String, JDK]  
keywords: Java, String, switch
description: Java7中所谓的支持switch字符串只是一个语法糖，底层还是一样：switch中只能使用整型，比如byte，short，char以及int。

---

支持switch字符串是Java7增加的一个新特性，那么它的底层是如何实现的呢？我们来看一个switch String的例子，然后分析编译器是如何处理的。

<!--more-->

## switch字符串的原理

switch String的语法示例如下：

```
/**
* Created by Alpha on Jan/21/15.
*/

public class StringSwitch {
  public static void main(String[] args) {
     String mode = args[0];
     switch (mode) {
        case "ACTIVE":
           System.out.println("Application is running on Active mode");
           break;
        case "PASSIVE":
           System.out.println("Application is running on Passive mode");
           break;
        case "SAFE":
           System.out.println("Application is running on Safe mode");
     }
  }
}
```

那么在底层是如何实现的呢？我们可以通过反编译看看编译器是如何处理上述代码的：

```

public class StringSwitch {
   public StringSwitch() {
   }

   public static void main(String[] args) {
       String mode = args[0];
       byte var3 = -1;
       switch(mode.hashCode()) {
       case -74056953:
           if(mode.equals("PASSIVE")) {
               var3 = 1;
           }
           break;
       case 2537357:
           if(mode.equals("SAFE")) {
               var3 = 2;
           }
           break;
       case 1925346054:
           if(mode.equals("ACTIVE")) {
               var3 = 0;
           }
       }

       switch(var3) {
       case 0:
           System.out.println("Application is running on Active mode");
           break;
       case 1:
           System.out.println("Application is running on Passive mode");
           break;
       case 2:
           System.out.println("Application is running on Safe mode");
       }

   }
}
```

可以看到处理流程是：

- 编译器首先调用字符串的`hashCode()`，返回一个int；
- 然后在对这个int进行switch；
- 每个switch case中再加上`equals()`比较来进行安全检查。


## 分析

我们看到实际上底层进行switch的是哈希值，所以Java7中所谓的支持switch字符串只是一个语法糖，底层还是一样：switch中只能使用整型，比如`byte`，`short`，`char`以及`int`。

另外switch case中通过`equals()`方法比较进行安全检查，这个检查是必要的，因为哈希可能会发生碰撞。

正式因为加上了这个检查，因此它的性能一定是不如使用枚举进行switch或者使用纯整数常量。

## 建议

因为上述性能问题，建议尽量使用纯整数常量进行swtich，或者用enum进行switch。

如果无法避免用字符串进行switch的话，还要注意大小写敏感的问题，建议统一用全大写字符串。
​
## Reference
http://javarevisited.blogspot.sg/2014/05/how-string-in-switch-works-in-java-7.html 






