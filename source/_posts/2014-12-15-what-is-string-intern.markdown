---
layout: post
title: "What's String.intern()"
date: 2014-12-15 15:23:45 +0800
comments: true
categories: [Java]
tags: [Java, JDK, String, Source Code]  
keywords: Java, String, intern
description: 分析String.intern()机制，及其使用场景。

---


Java语言中有8种基本类型和一种比较特殊的类型`String`。为了使这些类型在运行过程中速度更快，更节省内存，都提供了一种常量池的概念。常量池就类似一个Java系统级别提供的缓存。

其中8种基本类型的常量池都是系统协调的，而`String`类型的常量池比较特殊。它的主要使用方法有两种：

- 直接使用双引号声明出来的`String`对象会直接存储在常量池中。
- 如果不是用双引号声明的`String`对象，可以使用`String.intern()`方法。**intern方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中**

<!--more-->

## 源码

Jdk中源码如下：

```java
/** 
 * Returns a canonical representation for the string object. 
 * <p> 
 * A pool of strings, initially empty, is maintained privately by the 
 * class <code>String</code>. 
 * <p> 
 * When the intern method is invoked, if the pool already contains a 
 * string equal to this <code>String</code> object as determined by 
 * the {@link #equals(Object)} method, then the string from the pool is 
 * returned. Otherwise, this <code>String</code> object is added to the 
 * pool and a reference to this <code>String</code> object is returned. 
 * <p> 
 * It follows that for any two strings <code>s</code> and <code>t</code>, 
 * <code>s.intern()&nbsp;==&nbsp;t.intern()</code> is <code>true</code> 
 * if and only if <code>s.equals(t)</code> is <code>true</code>. 
 * <p> 
 * All literal strings and string-valued constant expressions are 
 * interned. String literals are defined in section 3.10.5 of the 
 * <cite>The Java&trade; Language Specification</cite>. 
 * 
 * @return  a string that has the same contents as this string, but is 
 *          guaranteed to be from a pool of unique strings. 
 */  
public native String intern();

```

这个方法是一个 native 的方法，但注释写的非常明了：
> 如果常量池中存在当前字符串，就会直接返回当前字符串；如果常量池中没有此字符串，会将此字符串放入常量池中后，再返回。



native方法的大体实现逻辑是：
JAVA 使用 jni 调用c++实现的`StringTable#intern()`方法, `StringTable#intern()`方法跟Java中的`HashMap`的实现是差不多的, 只是不能自动扩容。默认大小是1009。

要注意的是，String的String Pool是一个固定大小的`Hashtable`，默认值大小长度是1009，如果放进String Pool的String非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用`String.intern()`时性能会大幅下降（因为要一个一个找）。

在 jdk6中`StringTable`是固定的，就是1009的长度，**所以如果常量池中的字符串过多就会导致效率下降很快**。在jdk7中，`StringTable`的长度可以通过一个参数指定：
```
-XX:StringTableSize=99991
```


## Java6 和 Java7 下 intern 的区别
相信很多 JAVA 程序员都做做类似 `String s = new String("abc")`这个语句创建了几个对象的题目。 这种题目主要就是为了考察程序员对字符串对象的常量池掌握与否。
> 上述的语句中是创建了2个对象，第一个对象是"abc"字符串存储在常量池中，第二个对象在Heap中的 String 对象。



来看一段代码：


```
public static void main(String[] args) {
    String s = new String("1");
    s.intern();
    String s2 = "1";
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");
    s3.intern();
    String s4 = "11";
    System.out.println(s3 == s4);
}
```

打印结果是

- jdk6 下`false false`
- jdk7 下`false true`

然后将`s3.intern();`语句下调一行，放到`String s4 = "11";`后面。将`s.intern();` 放到S`tring s2 = "1";`后面：

```
public static void main(String[] args) {
    String s = new String("1");
    String s2 = "1";
    s.intern();
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");
    String s4 = "11";
    s3.intern();
    System.out.println(s3 == s4);
}
```

打印结果为：

- jdk6 下`false false`
- jdk7 下`false false`

### Java6

![user icon](/images/post/2014/12/string-intern-jdk6.png)

注：图中绿色线条代表 string 对象的内容指向。 黑色线条代表地址指向。

Java6中上述的所有打印都是 false 的，因为 **jdk6中的常量池是放在 Perm 区中的**，Perm 区和正常的 Heap 区域是完全分开的。

使用引号声明的字符串都是会直接在字符串常量池中生成，而 new 出来的 String 对象是放在 JAVA Heap 区域。所以拿一个Heap 区域的对象地址和字符串常量池的对象地址进行比较肯定是不相同的，即使调用`String.intern()`方法也是没有任何关系的。


### Java7

在 Java6 以及以前的版本中，字符串的常量池是放在堆的 Perm 区的，Perm 区是一个类静态的区域，主要存储一些加载类的信息，常量池，方法片段等内容，默认大小只有4m，一旦常量池中大量使用 intern 是会直接产生`java.lang.OutOfMemoryError: PermGen space`错误的。 

所以**在 jdk7 的版本中，字符串常量池已经从 Perm 区移到正常的 Heap 区域了**。

![user icon](/images/post/2014/12/string-intern-jdk7.png)

在第一段代码中，先看 s3和s4字符串。

- `String s3 = new String("1") + new String("1");`，这句代码中现在生成了2最终个对象，是字符串常量池中的“1” 和 Heap 中的 s3引用指向的对象。中间还有2个匿名的`new String("1")`我们不去讨论它们。此时s3引用对象内容是"11"，但此时常量池中是没有 “11”对象的。

- 接下来`s3.intern();`这一句代码，是将 s3中的“11”字符串放入 String 常量池中，因为此时常量池中不存在“11”字符串，因此常规做法是跟 jdk6 图中表示的那样，在常量池中生成一个 "11" 的对象，关键点是 jdk7 中常量池不在 Perm 区域了。所以常量池中不需要再存储一份对象了，可以直接存储堆中的引用。这份引用指向 s3 引用的对象。 也就是说引用地址是相同的。

- 最后`String s4 = "11";` 这句代码中"11"是显示声明的，因此会直接去常量池中创建，创建的时候发现已经有这个对象了，此时也就是指向 s3 引用对象的一个引用。所以 s4 引用就指向和 s3 一样了。因此最后的比较 `s3 == s4` 是 true。


再看 s 和 s2 对象。

-  `String s = new String("1");` 第一句代码，生成了2个对象。常量池中的“1” 和 Heap 中的字符串对象。`s.intern();` s 对象去常量池中寻找后发现 “1” 已经在常量池里了。 -- 这一点与s3/s4不一样。

- 接下来`String s2 = "1";` 这句代码是生成一个 s2的引用指向常量池中的“1”对象。 结果就是 s 和 s2 的引用地址明显不同。-- ？？？？？


![user icon](/images/post/2014/12/string-intern-jdk7.png)

来看第二段代码：

- 第一段代码和第二段代码的改变就是 `s3.intern();` 的顺序是放在`String s4 = "11";`后了。这样，首先执行`String s4 = "11";`声明 s4 的时候常量池中是不存在“11”对象的，执行完毕后，“11“对象是 s4 声明产生的新对象。然后再执行`s3.intern();`时，常量池中“11”对象已经存在了，因此 s3 和 s4 的引用是不同的。

- 第二段代码中的 s 和 s2 代码中，`s.intern();`，这一句往后放也不会有什么影响了，因为对象池中在执行第一句代码`String s = new String("1");`的时候已经生成“1”对象了。下边的s2声明都是直接从常量池中取地址引用的。 s 和 s2 的引用地址是不会相等的。


### 小结
从上述的例子代码可以看出** jdk7 版本对 intern 操作和常量池都做了一定的修改**。主要包括2点：

- 将String常量池 从 Perm 区移动到了 Java Heap区。


- 调用`String#intern()` 方法时，如果存在堆中的对象，会直接保存对象的引用，而不会重新创建对象。


## 使用场景

### 正例

如果用到了大量相同的String，那么可以使用`String#intern()`，例如：

```
static final int MAX = 1000 * 10000;
static final String[] arr = new String[MAX];

public static void main(String[] args) throws Exception {
    Integer[] DB_DATA = new Integer[10];
    Random random = new Random(10 * 10000);
    for (int i = 0; i < DB_DATA.length; i++) {
        DB_DATA[i] = random.nextInt();
    }
    long t = System.currentTimeMillis();
    for (int i = 0; i < MAX; i++) {
        //arr[i] = new String(String.valueOf(DB_DATA[i % DB_DATA.length]));
         arr[i] = new String(String.valueOf(DB_DATA[i % DB_DATA.length])).intern();
    }

    System.out.println((System.currentTimeMillis() - t) + "ms");
    System.gc();
}
```

运行的参数是：`-Xmx2g -Xms2g -Xmn1500M`

- 不使用 intern 的代码生成了1000w 个字符串，占用了大约640m 空间。 
- 使用了 intern 的代码生成了1345个字符串，占用总空间 133k 左右。

其实通过观察程序中只是用到了10个字符串，所以准确计算后应该是正好相差100w 倍。虽然例子有些极端，但确实能准确反应出** intern 使用后产生的巨大空间节省**。



另外，使用了 intern 方法后时间上有了一些增长。这是因为程序中每次都是用了 `new String` 后，然后又进行 intern 操作的耗时时间，这一点如果在内存空间充足的情况下确实是无法避免的，但我们平时使用时，内存空间肯定不是无限大的，**不使用 intern 占用空间导致 jvm 垃圾回收的时间是要远远大于这点时间的**。 


### 反例

上文提到：
在 jdk6中`StringTable`是固定的，就是1009的长度，**所以如果常量池中的字符串过多就会导致效率下降很快**。在jdk7中，`StringTable`的长度可以通过一个参数指定：
```
-XX:StringTableSize=99991
```

所以不能无用intern，把太多字符串放到常量池中。


## Reference
http://tech.meituan.com/in_depth_understanding_string_intern.html 