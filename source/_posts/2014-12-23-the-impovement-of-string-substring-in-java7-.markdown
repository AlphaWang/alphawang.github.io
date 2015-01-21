---
layout: post
title: "Is the Improvement of String.substring() in Java7 Really Reasonable? "
date: 2014-12-24 15:59:15 +0800
comments: true
categories: [Java]
tags: [Java, Source Code, JDK, String]  
keywords: Java, String, substring, List, sublist   
description: String#substring() 在Java6和Java7中的实现是不一样的。这是因为Java6的实现可能导致内存问题，所以Java7中为了改善这个问题修改了实现方式。那么Java7中的实现就真的合理吗？


---

`String#substring()`在Java6和Java7中的实现是不一样的。这是因为Java6的实现可能导致内存问题，所以Java7中为了改善这个问题修改了实现方式。那么Java7中的实现就真的合理吗？

首先让我们来猜测一下，Java是如何实现substring功能的。由于String是不可变的，可能我们会猜测实现机制如下图： 

![user icon](/images/post/2014/12/substring-user.png)

<!--more-->   

​然而，这个图并不完全正确，或者说并没有完全表示出Java堆中真正发生的事情。  



## Java6中的substring()

Java中字符串是通过字符数组来支持实现的，在JDK6中，String类包含3个实例变量：  
- `char[] value` 表示真实的字符数组；  
- `int offset` 表示数组的偏移量；  
- `int count` 表示String所包含的字符的个数。  

当调用`substring()`方法时，会创建一个新的字符串对象，但是这个字符串的值在java堆中仍然指向的是同一个数组，这两个字符串的不同之处只是他们的count和offset的值。

![java6 icon](/images/post/2014/12/substring-java6.png)


可以参考Java6中的源代码：

```java      
//Java 6
String(int offset, int count, char value[]) {
     this.value = value;
     this.offset = offset;
     this.count = count;
}
 
public String substring(int beginIndex, int endIndex) {
     //check boundary
     return new String(offset + beginIndex, endIndex - beginIndex, value);
}
```

 
### Java6中substring()可能导致的问题

这么实现有一个问题：如果你有一个非常长的字符串，但是你仅仅只需要这个字符串的一小部分，你需要的只是很小的部分，而这个子字符串却要包含整个字符数组。这可能导致内存溢出问题。 

我们可以用一个办法来规避这个问题：为`substring()`得到的子字符串重新创建一个对象。例如：
 
```
 String littleString = largeString.substring(0,2) + "";
```
或者： 
 
```
String littleString = new String(largeString.substring(0,2));
``` 
 
## Java7中的substring()

Java7中对上述问题做了修正，当调用`substring()`方法时，在堆中真正的创建了一个新的数组，当原字符数组没有被引用后就被GC回收了。

![java7 icon](/images/post/2014/12/substring-java7.png)
 
我们看源码：

```
// Java 7
    public String substring(int beginIndex, int endIndex) {
        return ((beginIndex == 0) && (endIndex == value.length)) ? this
                : new String(value, beginIndex, subLen);
    }
    
    public String(char value[], int offset, int count) {
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }    

```  

可以看到Java7通过`Arrays.copyOfRange`重新创建了一个字符数组。


## Java7的修改合理吗？

Java7虽然规避了substring可能出现的内存问题，但是新的实现真的好吗？

Java6的实现，当进行substring时，使用共享内容字符数组，速度会更快，不用重新申请内存。虽然有可能出现本文中的内存性能问题，但也是有方法可以解决的。

而Java7的实现，对任何String，即便不是Large String，都会重新申请内存，速度也会更慢，性能会更差。如果我们程序中处理的大部分都不是Large String的话，这种对性能的影响是不是得不偿失？

如果保持Java6的实现，在处理非Large String时，我们直接调用substring即可；而对Large String则用上文提到的规避方法来解决。

## List#sublist()的实现为什么没改变？

Java中有一个和`String#substring`有着类似逻辑、功能、实现机制的方法：`List#sublist`。Java6 处理Large List的sublist时，也会出现内存问题；而奇怪的时Java7并未对这个实现进行修改：

```
//AbstractList

    public List<E> subList(int fromIndex, int toIndex) {
        return (this instanceof RandomAccess ?
                new RandomAccessSubList<>(this, fromIndex, toIndex) :
                new SubList<>(this, fromIndex, toIndex));
    }

    SubList(AbstractList<E> list, int fromIndex, int toIndex) {
        l = list;
        offset = fromIndex;
        size = toIndex - fromIndex;
        this.modCount = l.modCount;
    }
```

所以我们在处理Large List时还是需要用规避方法：

```
	public static <E> List<E> sublist(List<E> originalList, int fromIndex, int toIndex) {
		return new ArrayList<E>(originalList.subList(fromIndex, toIndex));
	}

```


为什么Java7不对`List#sublist`做修改，以让它和`String#substring`的实现机制继续保持一致呢？不得而知。

## Reference
http://www.programcreek.com/2013/09/the-substring-method-in-jdk-6-and-jdk-7/ 

