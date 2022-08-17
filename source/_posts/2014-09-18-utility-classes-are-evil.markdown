---
layout: post
title: "Utility Classes Are Evil"
date: 2014-09-18 15:30:18 +0800
comments: true
categories: [Java]
tags: [Design Pattern, Java] 
keywords: Design Pattern, Java, Utility Classes, 工具类   
description: A utility class is a class filled with static methods. However, in an object-oriented world, utility classes are considered a very bad practice.  The use of utility classes to be an antipattern. More specifically, it violates common design principles   

---
This post is a summary of [this artical](http://www.javacodegeeks.com/2014/09/oop-alternative-to-utility-classes.html) and [this one](http://blogs.msdn.com/b/nickmalik/archive/2005/09/06/461404.aspx).

		
##What's Utility Classes  
A utility class is a class filled with static methods.  It is usually used to isolate a "useful" algorithm.  
>`StringUtils`, `IOUtils`, `FileUtils` from Apache Commons; `Iterables` and `Iterators` from Guava, and `Files` from JDK7 are perfect examples of utility classes.  

##Why Utility Classes  
If you have two classes `A` and `B`, and have a method `f()` that both must use, then the most naive approach is to repeat the function as a method in both classes. However, this violates the [Don't repeat yourself (DRY)][dry] approach to programming.   

[dry]:http://en.wikipedia.org/wiki/Don't_repeat_yourself  
    
  
  
The most natural solution is inheritance, but it's not always beneficial for `A` and `B` to be subclasses of some parent class. In my case, `A` was already a subclass of another class, while `B` was not. There was no way to make `A` and `B` subclasses of a parent class without breaking that other relationship.  

The alternative is to define the utility class: a public static class that sits in the global namespace, awaiting anyone to "borrow" them.   

They are not bad in themselves, but they do imply relationships between your data that are not explicitly defined. Also, if your static class has any static variables, then `A` and `B` never really know what they're getting into when they use it.

<!--more-->

##Disadvantage
However, in an object-oriented world, utility classes are considered a very bad practice.  
The use of utility classes to be an antipattern. More specifically, it violates common design principles: 

###Single Responsibility Principle
>A class should have one and only one reason to change  
  
You can design utility classes where all of the methods related to a single set of responsibilities.  That is entirely possible.  Therefore, I would note that this principle does not conflict with the notion of utility classes at all.  
That said, I've often seen helper classes that violate this principle.  They become "catch all" classes (or God Classes) that contain any method that the developer can't find another place for.  (e.g. a class containing a helper method for URL encoding, a method for looking up a password, and a method for writing an update to the config file... This class would violate the Single Responsibility Principle).

###Liskov Substitution Principle  
>Derived classes must be substitutable for their base classes  

This is kind of a no-op in that a utility class cannot have a derived class.  OK.  Does that mean that utility classes violate LSP?  I'd say not.  A helper class looses the advantages of OO completely, an in that sense, LSP doesn't matter... but it doesn't violate it.

###Interface Segregation Principle  
>Class interfaces should be fine-grained and client specific   

another no-op.  Since utility classes do not derive from an interface, it is difficult to apply this principle with any degree of seperation from the Single Responsibility Principle. 


###The Open Closed Principle  

>classes should be open for extension and closed for modification  

You cannot extend a utility class.  Since all methods are static, you cannot derive anything that extends from it.  In addition, the code that uses it doesn't create an object, so there is no way to create a child object that modifies any of the algorithms in a utility class.  They are all "unchangable".  

As such, a helper class simply fails to provide one of the key aspects of object oriented design: the ability for the original developer to create a general answer, and for another developer to extend it, change it, make it more applicable.  If you assume that you do not know everything, and that you may not be creating the "perfect" class for every person, then utility classes will be an anathema to you.

###The Dependency Inversion Principle  
>Depend on abstractions, not concrete implementations  

This is a simple and powerful principle that produces more testable code and better systems.  If you minimize the coupling between a class and the classes that it depends upon, you produce code that can be used more flexibly, and reused more easily.  

However, a utility class cannot participate in the Dependency Inversion Principle.  It cannot derive from an interface, nor implement a base class.  No one creates an object that can be extended with a helper class.  This is the "partner" of the Liskov Substitution Principle, but while utility classes do not violate the LSP, they do violate the DIP. 

---

**In summary, utility classes are not proper objects; therefore, they don’t fit into object-oriented world. They were inherited from procedural programming, mostly because most were used to a functional decomposition paradigm back then.**

And there are other aritcals about this topic:  [Are Helper Classes Evil?][1] by Nick Malik, [Why helper, singletons and utility classes are mostly bad][2] by Simon Hart, [Avoiding Utility Classes][3] by Marshal Ward, [Kill That Util Class!][4] by Dhaval Dalal, [Helper Classes Are A Code Smell][5] by Rob Bagby.
[1]: http://blogs.msdn.com/b/nickmalik/archive/2005/09/06/461404.aspx
[2]: http://smart421.wordpress.com/2011/08/31/why-helper-singletons-and-utility-classes-are-mostly-bad-2/
[3]: http://www.marshallward.org/avoiding-utility-classes.html
[4]: http://www.jroller.com/DhavalDalal/entry/kill_that_util_class
[5]: http://www.robbagby.com/posts/helper-classes-are-a-code-smell/

##Object-Oriented Alternative

###Example1
Let's take `NumberUtils` for example:  

```
// This is a terrible design, don't reuse
public class NumberUtils {
  public static int max(int a, int b) {
    return a > b ? a : b;
  }
}
```

In an object-oriented paradigm, we should instantiate and compose objects, thus letting them manage data when and how they desire. Instead of calling supplementary static functions, we should create objects that are capable of exposing the behaviour we are seeking:  

```
public class Max implements Number {
  private final int a;
  private final int b;
  public Max(int x, int y) {
    this.a = x;
    this.b = y;
  }
  @Override
  public int intValue() {
    return this.a > this.b ? this.a : this.b;
  }
}
```

This procedural call:  
```
int max = NumberUtils.max(10, 5);  
```  
Will become object-oriented:  
```
int max = new Max(10, 5).intValue();
```

###Example2  
Say, for instance, you want to read a text file, split it into lines, trim every line and then save the results in another file. This is can be done with `FileUtils` from Apache Commons:

```
void transform(File in, File out) {
  Collection<String> src = FileUtils.readLines(in, "UTF-8");
  Collection<String> dest = new ArrayList<>(src.size());
  for (String line : src) {
    dest.add(line.trim());
  }
  FileUtils.writeLines(out, dest, "UTF-8");
}
```   
The above code may look clean; however, this is procedural programming, not object-oriented. We are manipulating data (bytes and bits) and explicitly instructing the computer from where to retrieve them and then where to put them on every single line of code. We’re defining a procedure of execution.  
The OO alternative is:  

```
void transform(File in, File out) {
  Collection<String> src = new Trimmed(
    new FileLines(new UnicodeFile(in))
  );
  Collection<String> dest = new FileLines(
    new UnicodeFile(out)
  );
  dest.addAll(src);
}
```  

`FileLines` implements `Collection<String>` and encapsulates all file reading and writing operations. An instance of `FileLines` behaves exactly as a collection of strings and hides all I/O operations. When we iterate it — a file is being read. When we addAll() to it — a file is being written.

`Trimmed` also implements `Collection<String>` and encapsulates a collection of strings (Decorator pattern). Every time the next line is retrieved, it gets trimmed.

All classes taking participation in the snippet are rather small: `Trimmed`, `FileLines`, and `UnicodeFile`. **Each of them is responsible for its own single feature, thus following perfectly the single responsibility principle.**

On our side, as users of the library, this may be not so important, but for their developers it is an imperative. It is much easier to develop, maintain and unit-test class `FileLines` rather than using a readLines() method in a 80+ methods and 3000 lines utility class `FileUtils`. Seriously, look at its source code.

###Lazy Execution
An object-oriented approach enables lazy execution. The in file is not read until its data is required. If we fail to open out due to some I/O error, the first file won’t even be touched. The whole show starts only after we call addAll().

All lines in the second snippet, except the last one, instantiate and compose smaller objects into bigger ones. This object composition is rather cheap for the CPU since it doesn’t cause any data transformations.

Besides that, it is obvious that the second script runs in O(1) space, while the first one executes in O(n). This is the consequence of our procedural approach to data in the first script.

In an object-oriented world, there is no data; there are only objects and their behavior!




##References
http://www.javacodegeeks.com/2014/09/oop-alternative-to-utility-classes.html  
http://blogs.msdn.com/b/nickmalik/archive/2005/09/06/461404.aspx  
http://www.marshallward.org/avoiding-utility-classes.html  

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

