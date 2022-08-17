---
layout: post
title: "Learning JDK Source Code: Negative Examples"
date: 2014-12-01 15:00:45 +0800
comments: true
categories: [Java]
tags: [Java, JDK, Source Code]  
keywords: Java, Design Patterns
description: 本文记录了一些Java源码中设计不合理之处、违反设计模式之处，以及其他可以改进的地方。

---

本文记录了一些Java源码中设计不合理之处、违反设计模式之处，以及其他可以改进的地方。

<!--more-->


# 1. 构造对象语法

## 参数化类型的构造函数啰嗦

如果你调用参数化类的构造函数，那么你必须要指定类型参数，即便上下文中已明确了类型参数。这通常要求你连续两次提供类型参数：

```
Map <String, List <String>> m = new HashMap<String, List<String>>(); 

```


而假设HashMap提供了如下静态工厂：

```
public static  <K, V> HashMap <K, V> newInstance(){    
   return new HashMap<K, V>();    
}
```

那么你就可以讲上文冗长的声明替换为如下这种简洁的形式：

```
Map < String, List < String >> m = HashMap.newInstance();
```

补充1：`com.google.common.collect.Lists`可以解决这个问题：

```
List < String > l = Lists.newArrayList();  
 
     public   static  ArrayList  newArrayList ()
    {
         return   new  ArrayList(); 
    }
```

补充2：Java7做了优化，可以这样声明：

```
Map <String, List <String>> m = new HashMap<>();
```


# 2. 创建了不必要的对象

## Boolean(String)

`Boolean(String)`有点多余，因为已经有**静态工厂方法**：`Boolean.valueOf(String)`，它比Boolean(String)更可取。

构造函数每次被调用时都会创建一个新对象，而静态工厂方法则从来不要求这样做，实际上也不会这么做。

```
    public Boolean(String s) {
        this(toBoolean(s));
    }
    private static boolean toBoolean(String name) {
        return ((name != null) && name.equalsIgnoreCase("true"));
    }
 
    public static Boolean valueOf(String s) {
        return toBoolean(s) ? TRUE : FALSE;
    }
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);
```



# 3. 误用finalize

要避免用finalize来释放资源，而**应该提供一个显式的终止方法**。例如FileInputStream.close();

**finalizer的作用之一是，可以充当“安全网”，以防对象所有者忘记调用显式的终止方法**。虽然不能保证finalizer会被及时调用，但当客户端没有调用显式终止方法时，迟一点释放资源总比不释放好。不过如果finalizer发现有未被终止的资源，则必须打印一条警告，表明客户端代码有bug，需要修复。

唯一声称保证finalizer()会被执行的方法是`System.runFinalizersOnExit`，以及`Runtime.runFinalizersOnExit`。
> 但这两个方法都有致命缺陷并且都已弃用。
> 

## FileInputStream.finalize()

JDK有四个类（`FileInputStream`、`FileOutputStream`、`Timer`、`Connection`）使用了finalizer作为安全网，以防显式终止方法未被调用。不幸的是，这些finalizer都没有打印警告。当API发布后，这种警告一般就不能添加到API了，因为可能破坏已有的客户端代码。

```
    /**
     * Ensures that the <code>close</code> method of this file input stream is
     * called when there are no more references to it.
     *
     * @exception  IOException  if an I/O error occurs.
     * @see        java.io.FileInputStream#close()
     */
    protected void finalize() throws IOException {
        // should log an error!
        if ((fd != null) &&  (fd != FileDescriptor.in)) {
            try {
                close();
            } finally {
            }
        }
    }  
```


# 4. 违反equals约定

## URL.equals()

`java.net.UR`L的equals方法依赖于对URL中主机的IP地址的比较，而将主机名转译成IP地址需要访问网络，随着时间推移，并不保证能返回相同的结果。

——**违反一致性**。这就会导致URL的equals方法违反约定，并且已经在实践中引起问题了。

> 不幸的是，由于兼容性需求，这一行为无法改变。除了少数例外情况，equals方法必须对驻留在内存中的对象进行确定性计算。

## Timestamp.euals()

`java.sql.Timestamp`扩展了`java.util.Date`类并增加了`nanoseconds`字段。其equals方法**违反了对称性**：如果Timestamp和Date被用于同一个集合中，或以其他什么方式混在一起使用，则会引起错误的行为。

> 无法在扩展可实例化类（即非抽象类）的时候，增加一个值组件，同时又保证equals约定。

`Timestamp`有一个免责声明，提醒程序员不要混用Date和Timestamp。虽然只要不混用他们就不会有麻烦，但是谁都不能阻止你混用他们，而结果导致的错误将会很难调试。

```
    /**
     *
     * Note: This method is not symmetric with respect to the
     * <code>equals(Object)</code> method in the base class.
     *
     */
    public boolean equals(java.lang.Object ts) {
      if (ts instanceof Timestamp) {
        return this.equals((Timestamp)ts);
      } else {
        return false;
      }
    }  
```


# 5. compareTo与equals不一致

compareTo方法的等同性测试必须与equals方法的结果相同。如果遵守了这一条，则称compareTo方法所施加的顺序与equals一致；反之则称为与equals不一致。

当然与equals不一致的compareTo方法仍然是可以工作的。但是，如果一个有序集合包含了该类的元素，则这个集合可能就不能遵守相应集合接口（Collection、Set、Map）的通用约定。这是因为**这些集合接口的通用约定是基于equals方法的，但是有序集合却使用了compareTo而非equals来执行等同性测试**。

## BigDecimal.compareTo()

BigDecimal类的compareTo方法与equals不一致：

- 如果创建一个`HashSet`实例，并添加两个元素`new BigDecimal("1.0")`和`new BigDecimal("1.00")`，则集合会包含两个元素，因为这两个实例通过equals检测并不相等；
- 而如果使用`TreeSet`而非HashSet，则集合中会只包含一个元素，因为这两个实例通过compareTo检测是相等的。


```
public int compareTo(BigDecimal val) {
   // Quick path for equal scale and non-inflated case.
   if (scale == val.scale) {
       long xs = intCompact;
       long ys = val.intCompact;
       if (xs != INFLATED && ys != INFLATED)
           return xs != ys ? ((xs > ys) ? 1 : -1) : 0;
   }
   int xsign = this.signum();
   int ysign = val.signum();
   if (xsign != ysign)
       return (xsign > ysign) ? 1 : -1;
   if (xsign == 0)
       return 0;
   int cmp = compareMagnitude(val);
   return (xsign > 0) ? cmp : -cmp;
}


public boolean equals(Object x) {
   if (!(x instanceof BigDecimal))
       return false;
   BigDecimal xDec = (BigDecimal) x;
   if (x == this)
       return true;
   if (scale != xDec.scale)
       return false;
   long s = this.intCompact;
   long xs = xDec.intCompact;
   if (s != INFLATED) {
       if (xs == INFLATED)
           xs = compactValFor(xDec.intVal);
       return xs == s;
   } else if (xs != INFLATED)
       return xs == compactValFor(this.intVal);

   return this.inflate().equals(xDec.inflate());
}
```


# 6. 接口设计问题

## Cloneable接口

Cloneable接口的目的是作为对象的一个mixin接口，表明对象允许克隆；但这个目的没有达到。

其主要缺点是，Cloneable缺少一个`clone()`方法，而`Object.clone()`是受保护的。

通常，实现接口是为了表明类可以为它的客户做些什么；而**Cloneable却改变了超类中受保护方法的行为**。

> ——区别java.rmi.Remote接口，其中也不具有任何方法，它是一个记号接口。



# 7. public类不应暴露其内部字段

如果一个类可以被包外访问，那么就要提供访问方法，以便可以灵活地改变类的内部表示。如果public类暴露了其数据域，那就不能在将来改变内部表示了。
## Point

```
public class Point extends Point2D implements java.io.Serializable {
    /**
     * The X coordinate of this <code>Point</code>.
     * If no X coordinate is set it will default to 0.
     */
    public int x;
    /**
     * The Y coordinate of this <code>Point</code>.
     * If no Y coordinate is set it will default to 0.
     */
    public int y;
```

## Dimension

```
public class Dimension extends Dimension2D implements java.io.Serializable {
    /**
     * The width dimension; negative values can be used.
     */
    public int width;
    /**
     * The height dimension; negative values can be used.
     */
    public int height;
```


# 8. 不可变类的设计

## 不可变类无需提供拷贝构造函数: String(String  original)：

不可变对象可以自由共享，所以无需进行保护性拷贝。实际上你根本无需做任何拷贝，**因为这些拷贝始终与源对象相等**。因此，**你不需要，也不应该为不可变类提供clone方法或者拷贝构造函数**。

【反例】这一点在Java平台早期并没有被很好地理解，导致String类具有拷贝构造函数，应该尽量不去用这个函数。

```
    /**
     * Initializes a newly created {@code String} object so that it represents
     * the same sequence of characters as the argument; in other words, the
     * newly created string is a copy of the argument string. Unless an
     * explicit copy of {@code original} is needed, use of this constructor is
     * unnecessary since Strings are immutable.
     *
     * @param  original
     *         A {@code String}
     */
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
```
## 不可变类应该不能继承: BigInteger / BigDecimal：

当BigInteger和BigDecimal编写出来时，对于“**不可变类实际上必须final**”并没有得到广泛的理解，所以这两个类的方法都可以被重写。不幸的是，为了保持向后兼容，这个问题一直没有得到修正。

如果你编写的类的安全性依赖于（来自不可信客户端的）BigInteger或BigDecimal的不可变性，那么就必须检查参数是真正的BigInteger/BigDecimal，还是不可信任的子类实例。如果是后者，你必须把它当成是可变的，并进行保护性拷贝：

```
public static BigInteger safeInstance(BigInteger val) {
   if (val.getClass() != BigInteger.class )
          return new BigInteger(val.toByteArray());
   return val;
```

## Date,  Point：理应设计成不可变类

应该永远让小的值对象不可变，例如PhoneNumber、Complex。
Java平台库中有许多这种类，例如`java.util.Date`、`java.awt.Point`，它们理论上应当是不可变的，但实际上却是可变的。

## EnumSet理应不可变

 

 


# 9. 异常设计

## finally中应该禁止return

It is especially difficult to understand the behavior of a program that executes a break, continue, or return statement in a Try block only to have the statement's behavior vetoed by a finally block.

**Never exit a finally block with a return, break, continue, or throw, and never allow a checked exception to propagate out of a finally block.**

```
    static boolean decision() { 
        try { 
            return true; 
        } finally { 
            return false; 
        } 
    }
```

这个程序返回false。无论try块是否正常执行完，finnaly都会被执行。

## finally中应该禁止抛出异常

另外，finally中应该禁止抛出异常。否则finally中剩下的语句就不会执行，破坏逻辑。

## 继承方法抛出的异常

The set of checked exceptions that a method can throw is the **intersection** of the sets of checked exceptions that it is declared to throw in all applicable types, **not the union**.

```
interface Type1 { 
     void f() throws CloneNotSupportedException; 
}
 
interface Type2 { 
     void f() throws InterruptedException; 
}
 
interface Type3 extends Type1, Type2 { 
}
 
public class Arcane3 implements Type3 {
 
     public void f() { 
        System.out.println( "Hello world" ); 
    }
 
     public static void main(String[] args) { 
        Type3 t3 = new Arcane3(); 
         t3.f();   // 不抛异常！！！
    }
 
}
```

# 10. 类型转换

## int相乘可能溢出

两个int相乘，得到的还是int，这就可能溢出！

```
long MICROS_PER_DAY = 24 * 60 * 60 * 1000 * 1000; //不精确
long MICROS_PER_DAY = 24 L * 60 * 60 * 1000 * 1000; //精确
```

应该自动切换到更大的类型，以避免溢出；
或者直接抛出Exception，都比溢出要好。

## mixed-type computations

如果遇到跨类型计算，jdk会把低类型自动提升为高类型，然后再计算。但这种转换有时会导致问题。例如：

```
Long.toHexString(0x100000000L + 0xcafebabe); //cafebabe
```

因为是long + int，所以后面的int会自动提升为long，再计算。即被提升为0xffffffffcafebabeL。

```
  0xffffffffcafebabeL
+ 0x0000000100000000L 
= 0x00000000cafebabeL
```

所以得到cafebase，而不是想象中的1cafebabe。
我们得出教训：**跨类型计算可能带来混淆，所以要坚决避免！**上例可以改为如下：

```
Long.toHexString(0x100000000L + 0xcafebabeL); //1cafebabe
```

> Negative hex literals appear positive。十六进制的字面值，其最高位代表正负。

## 三元运算符的操作数类型
要注意三元运算符，它没有要求第二和第三个操作数类型一致。

1. 如果类型都为T；则结果为T。
2. 如果其中一个类型T为byte/shor/char，另一个是int常量；则结果为T。
3. 如果为其他情况，则结果为提升类型。

```
char x = 'X';
int i = 0;
System.out.print(false ? i : x); //输出88，结果为int类型
```

理应强制要求他们类型一致。

##+=和-=的自动类型转换：损失精度

`+=`、`-=`等运算符会自动类型转换，即将计算结果自动转换为左侧操作数的类型。这有时会导致意想不到的问题。
例如：

```
short x = 0; 
int i = 123456;
x += i; //自动转换为short，损失精度，但不报错
x = x + i; // Won't compile - "possible loss of precision"
```
应该不要做自动类型转换，以编译报错提醒用户。（与第二句普通赋值语句保持一致）

- ——另外一个问题，`+=`左侧不能为Object，例如：

```
Object x = "Buy "; 
String i = "Effective Java!";
x = x + i;  //合法
x += i;     //非法
```

- ——Narrowing Primitive Conversion
An unfortunate fact about the compound assignment operators is that they can silently perform narrowing primitive conversions , which are conversions from one numeric type to a less expressive numeric type.

```
short i = -1;

while (i != 0)
    i >>>= 1;
```
1. 初始值为0xffff；  
2. 执行>>>=时，先将其提升为int，变为0xffffffff；  
3. 接着移位，变为0x7fffffff；  
4. 接着赋值回i，这时会执行从int到short的转换，变为0xffff

所以上例是一个无限循环。


# 11. 语言设计

## long字面值可以用小写L

用小写L容易与数字1混淆！
应该强制用大写L，小写L非法。

## 重写toString()

```
Object numbers = new char[] { '1', '2', '3' };
System.out.println(numbers); // [C@16f0472
```
貌似Array应该默认重写toString方法。

## 不能静态导入Arrays.toString()

为了解决上一个问题，你可能想静态导入Arrays.toString()，然后调用：

```
toString(numbers);
```
但是会编译报错，编译器去查找当前类的toString()方法，发现参数不匹配。。

## String(byte[])依赖于默认字符集

String(byte[])的文档说明，它依赖于默认字符集：
> Constructs a new String by decoding the specified byte array **using the platform's default charset**. The length of the new String is a function of the charset, and hence may not be equal to the length of the byte array. The behavior of this constructor when the given bytes are not valid in the default charset is unspecified。

但是JRE的默认字符集依赖于操作系统和locale。所以，it was not such a good idea to provide a String(byte[]) constructor that depends on the default charset: 

```
String str = new String(bytes, "ISO-8859-1");
```

## final字段与final方法的含义完全不同


## Thread.run()不应该公开

- Thread.run()是一个public方法，很可能导致被误调用——想调start()，结果却调了run() 。
- If `Thread` didn't have a public `run` method, it would be impossible for programmers to invoke it accidentally. 
- The `Thread` class has a public `run` method because it implements `Runnable`, but it didn't have to be that way. 
- An alternative design would be for each `Thread` instance to encapsulate a `Runnable`, giving rise to composition in place of interface inheritance.


## 方法名不合理

methods should have names that describe their primary functions. Given the behavior of `Thread.interrupted`, it should have been named `clearInterruptStatus`.


```
/**
* Tests if some Thread has been interrupted. The interrupted state
* is reset or not based on the value of ClearInterrupted that is
* passed.
*/

private native boolean isInterrupted( boolean ClearInterrupted );

public static boolean interrupted() {
  return currentThread().isInterrupted( true );
}

public boolean isInterrupted() {
  return isInterrupted( false );
}
```

## shadow local variables

perhaps it makes sense to forbid shadowing of type parameters, in the same way that shadowing of local variables is forbidden.



未完待续。。。