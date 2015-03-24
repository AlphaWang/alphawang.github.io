---
layout: post
title: "Learning JDK Source Code: Positive Examples"
date: 2014-12-02 16:21:09 +0800
comments: true
categories: [Java]
tags: [Java, JDK, Source Code]  
keywords: Java, Design Patterns
description: 本文记录了一些Java源码中的良好编程习惯、运用的设计模式。

---

本文记录了一些Java源码中的良好编程习惯、运用的设计模式。

<!--more-->

#编程习惯

## 1. 用工厂方法替代构造函数

### Boolean.valueOf()

通过一个boolean简单类型，构造Boolean对象引用。

> **优点：无需每次被调用时都创建一个新对象。同时使得类可以严格控制在哪个时刻有哪些实例存在 **


```java
    /**
     * Returns a <code>Boolean</code> with a value represented by the
     * specified string.  The <code>Boolean</code> returned represents a
     * true value if the string argument is not <code>null</code>
     * and is equal, ignoring case, to the string {@code "true"}.
     *
     * @param   s   a string.
     * @return  the <code>Boolean</code> value represented by the string.
     */
    public static Boolean valueOf(String s) {
        return toBoolean(s) ? TRUE : FALSE;
    }
```
 
静态工厂方法Boolean.valueOf(String)几乎总是比构造函数Boolean(String)更可取。构造函数每次被调用时都会创建一个新对象，而静态工厂方法则从来不要求这样做，实际上也不会这么做。


### BigInteger.probablePrime()

构造方法`BigInteger(int, int, Random)`返回一个可能为素数的BigInteger，而用一个名为`BigInteger.probablePrime()`的静态工厂方法就更好。（JDK1.4最终增加了这个方法。）

> **优点：方法名对客户端更友好**

```
public class BigInteger extends Number implements Comparable<BigInteger> { 
   /**
     * Returns a positive BigInteger that is probably prime, with the
     * specified bitLength. The probability that a BigInteger returned
     * by this method is composite does not exceed 2<sup>-100</sup>.
     *
     */
    public static BigInteger probablePrime(int bitLength, Random rnd) {
    if (bitLength < 2)
        throw new ArithmeticException("bitLength < 2");
        // The cutoff of 95 was chosen empirically for best performance
        return (bitLength < SMALL_PRIME_THRESHOLD ?
                smallPrime(bitLength, DEFAULT_PRIME_CERTAINTY, rnd) :
                largePrime(bitLength, DEFAULT_PRIME_CERTAINTY, rnd));
    }
```

### EnumSet

JDK1.5引入的`java.util.EnumSet`类没有public构造函数，只有静态工厂方法。根据底层枚举类型的大小，这些工厂方法可以返回两种实现：

- 如果拥有64个或更少的元素（大多数枚举类型都是这样），静态工厂方法返回一个`RegularEnumSet`实例，用单个`long`来支持；
- 如果枚举类型拥有65个或更多的元素，静态工厂方法则返回`JumboEnumSet`实例，用`long数组`来支持。

> **优点：静态工厂方法能返回任意子类型的对象。可以根据参数的不同，而返回不同的类型。**

```
public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E>
    implements Cloneable, java.io.Serializable
{
    /**
     * Creates an empty enum set with the specified element type.
     *
     * @param elementType the class object of the element type for this enum
     *     set
     * @throws NullPointerException if <tt>elementType</tt> is null
     */
    public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");
        if (universe.length <= 64)
            return new RegularEnumSet<E>(elementType, universe);
        else
            return new JumboEnumSet<E>(elementType, universe);
    }
}
 
class RegularEnumSet<E extends Enum<E>> extends EnumSet<E> {
}
 
class JumboEnumSet<E extends Enum<E>> extends EnumSet<E> {
}
 ```
 

### Collections.unmodifiableMap(Map)

Java集合框架中有32个集合接口的便利实现，提供不可修改的集合、同步集合等等。几乎所有的实现都通过一个不可实例化类（`java.util.Collections`）中的静态工厂方法导出，返回对象的类都是非public的。

> **优点：静态工厂方法能返回任意子类型的对象。可以返回一个对象而无需使相应的类public。用这种方式隐藏实现类能够产生一个非常紧凑的API**

```
public class Collections {
    /**
     * Returns an unmodifiable view of the specified map.  This method
     * allows modules to provide users with "read-only" access to internal
     * maps.  Query operations on the returned map "read through"
     * to the specified map, and attempts to modify the returned
     * map, whether direct or via its collection views, result in an
     * <tt>UnsupportedOperationException</tt>.<p>
     *
     */
    public static <K,V> Map<K,V> unmodifiableMap(Map<? extends K, ? extends V> m) {
        return new UnmodifiableMap<K,V>(m);
    }
 
    private static class UnmodifiableMap<K,V> implements Map<K,V>, Serializable {
      }
```

## 2. 私有化构造函数，实现Singleton

### Arrays

这种工具类设计出来并不是为了实例化它。然而，如果不显式地编写构造函数，编译器则会提供一个公共的无参数的默认构造方法。
所以将构造函数私有化：

```
public class Arrays {
    // Suppresses default constructor, ensuring non-instantiability.
    private Arrays() {
    }
 ```

当然，还可以在这个私有构造器内部加上 `throw new AssertionError()`，可以确保该方法不会再类内部被意外调用。

**这种习惯用法的副作用是类不能被子类化了**。子类的所有构造函数必须首先隐式或显式地调用父类构造函数，而在这种用法下，子类就没有可访问的父类构造函数可调用了。

### TimeUnit

`java.util.concurrent.TimeUnit`使用枚举来实现Singleton：

```
public enum TimeUnit {
    MILLISECONDS {
        public long toNanos(long d)   { return x(d, C2/C0, MAX/(C2/C0)); }
        public long toMicros(long d)  { return x(d, C2/C1, MAX/(C2/C1)); }
        public long toMillis(long d)  { return d; }
        public long toSeconds(long d) { return d/(C3/C2); }
        public long toMinutes(long d) { return d/(C4/C2); }
        public long toHours(long d)   { return d/(C5/C2); }
        public long toDays(long d)    { return d/(C6/C2); }
        public long convert(long d, TimeUnit u) { return u.toMillis(d); }
        int excessNanos(long d, long m) { return 0; }
    },
    SECONDS {
       .....
    },
    MINUTES {
       .....
    },
    HOURS {
       ......
    },
    DAYS {
       .....
    };

    // TimeUnit.sleep()用来替代Thread.sleep()
    public void sleep(long timeout) throws InterruptedException {
        if (timeout > 0) {
            long ms = toMillis(timeout);
            int ns = excessNanos(timeout, ms);
            Thread.sleep(ms, ns);
        }
    }
```

## 3. 避免创建不必要的对象

### Map.keySet()


Map接口的keySet()方法返回Map对象的一个Set视图，包含该Map的所有key。
看起来好像每次调用keySet()都需要创建一个新的Set实例。而实际上，虽然返回的Set通常是可变的，但返回的对象在功能上是等同的：**如果其中一个返回对象改变，其他对象也会改变，因为他们的底层都是同一个Map实例**。虽然创建多个KeySet视图对象并没有害处，但也没有必要。

```
public abstract class AbstractMap<K,V> implements Map<K,V> {
 
    /**
     * Each of these fields are initialized to contain an instance of the
     * appropriate view the first time this view is requested.  The views are
     * stateless, so there's no reason to create more than one of each.
     */
   transient volatile Set<K>        keySet = null;
   public Set<K> keySet() {
    if (keySet == null) {
        keySet = new AbstractSet<K>() {
        .....
        };
    }
    return keySet;
    }    
}   
 ```
注意在构造keySet之前 对齐进行了null检查；只有当它是null时才会初始化。



## 4. 消除无用的对象引用

### LinkedHashMap.removeEldestEntry()


缓存实体的生命周期不容易确定，随着时间推移，实体的价值越来越低。在这种情况下，缓存应该不定期地清理无用的实体。可以通过一个后台线程来清理（可能是`Timer`或`ScheduledThreadPoolExecutor`），也可以在给缓存添加新实体时进行清理。

LikedHashMap可利用其removeEldestEntry，删除较老的实体：

```
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{
   /**
     * It causes newly allocated entry to get inserted at the end of the linked list and
     * removes the eldest entry if appropriate.
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
        createEntry(hash, key, value, bucketIndex);
        // Remove eldest entry if instructed, else grow capacity if appropriate
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        } else {
            if (size >= threshold)
                resize(2 * table.length);
        }
    }
    /**
     * Returns <tt>true</tt> if this map should remove its eldest entry.
     * This method is invoked by <tt>put</tt> and <tt>putAll</tt> after
     * inserting a new entry into the map.  
     *
     * Sample use: 
     * <pre>
     *     private static final int MAX_ENTRIES = 100;
     *
     *     protected boolean removeEldestEntry(Map.Entry eldest) {
     *        return size() > MAX_ENTRIES;
     *     }
     * </pre>
     *
     */
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }

```
    
——可以继承LinkedHashMap，覆盖其removeEldestEntry方法。


> 注：如果想要缓存中的对象只要不被引用，就自动清理；则可以用WeakHashMap

## 5. 如果指定了toString返回值的格式，则应该提供一个对应的静态工厂方法或构造函数


### BigInteger.toString()


```   
    /**
     * Returns the String representation of this BigInteger in the
     * given radix.  
     *
     */
    public String toString(int radix) {
 
    /**
     * Returns the decimal String representation of this BigInteger.
     */
    public String toString() {
        return toString(10);
    }

```
 
对应的构造函数如下。这样程序员能容易地在对象及其字符串表示之间来回转换

```
    /**
     * Translates the decimal String representation of a BigInteger into a
     * BigInteger.  
     */
    public BigInteger(String val) {
        this(val, 10);
    }
    /**
     * Translates the String representation of a BigInteger in the specified
     * radix into a BigInteger.  
     */
    public BigInteger(String val, int radix) { 
```

## 6. 不可变对象

### 共享不可变对象内部信息:  BigInteger.negate()：


> 我们可以自由地共享不可变对象，还可以共享他们的内部信息。

【例】BigInteger类内部使用了一个符号数值表示法（sign-magnitude representation），符号用一个int表示，数值则用一个int数组表示。`negate()`方法会创建一个数值相同但符号相反的新`BigInteger`，该方法不需要拷贝数组，新创建的BigInteger只需要指向源对象中的数组即可。

```
    /**
     * Returns a BigInteger whose value is {@code (-this)}.
     *
     * @return {@code -this}
     */
    public BigInteger negate() {
          return new BigInteger(this.mag, -this.signum);
    }
```
 
### 鼓励重用现有不可变实例: BigInteger.ZERO

**不可变对象生来就是线程安全的，他们不需要同步**。当多个线程并发访问不可变对象时，他们不会遭到破坏。这无疑是实现线程安全的最容易的方法。实际上，不会有线程能观察到其他线程对不可变对象的影响。所以不可变对象可以被自由地共享。

不可变类应当利用这种优势，鼓励客户端尽可能重用现有实例。一个简单的方法是为常用的值提供public static final的常量。


```
    public static final BigInteger ZERO = new BigInteger(new int[0], 0);
    public static final BigInteger ONE = valueOf(1);
    public static final BigInteger TEN = valueOf(10);
```

### 为不可变类提供companion class:  String vs. StringBuilder

不可变类的一个缺点是，对于每个不同的值都需要一个单独的对象。那么在执行复杂的多步操作时，每一步都会创建一个新的对象。

通过提供`companion class`可以解决这个问题。例如当需要对`String`执行复杂操作时，建议使用`StringBuilder`。


### 不可变类中可以有nonfinal域，用于存储缓存: String.hashCode()：

不可变类所有域必须是final的。实际上这些规则比较强硬，为了提供性能可以有所放松。实际上应该是没有方法能够对类的状态产生外部可见的改变（no method may produce an externally visible change in the object’s state）。

然而，一些不可变类拥有一个或多个nonfinal域，用于缓存昂贵计算的结果。这个技巧可以很好地工作，因为对象是不可变的，保证了相同的计算总是返回同样的结果。

```
    /** Cache the hash code for the string */
    private int hash; // Default to 0
 
    public int hashCode() {
      int h = hash;
      int len = count;
      if (h == 0 && len > 0) {
        int off = offset;
        char val[] = value;
            for (int i = 0; i < len; i++) {
                h = 31*h + val[off++];
            }
            hash = h;
      }
      return h;
    }
```
 

# 设计模式

## 1. Template Method


### Arrays.sort()

```
// Arrays  
   public static void sort(Object[] a) {  
        ...  
        ComparableTimSort.sort(a);  
    }  
  
// ComparableTimSort  
   static void sort(Object[] a, int lo, int hi) {  
        ...  
        binarySort(a, lo, hi, lo + initRunLen);  
             
   } 
```

``` 
    //算法框架  
    private static void binarySort(Object[] a, int lo, int hi, int start) {  
        assert lo <= start && start <= hi;  
        if (start == lo)  
            start++;  
        for ( ; start < hi; start++) {  
            @SuppressWarnings("unchecked")  
            Comparable<Object> pivot = (Comparable) a[start];  
  
            // Set left (and right) to the index where a[start] (pivot) belongs  
            ....  
            while (left < right) {  
                int mid = (left + right) >>> 1;  
                if (pivot.compareTo(a[mid]) < 0) //compareTo这个算法步骤，是由各个Comparable的子类定义的  
                    right = mid;  
                else  
                    left = mid + 1;  
            }  
            ....  
        }  
    } 
``` 

Q：这个算法框架并不是设计在父类中，而是在一个工具类中

A：是的，与教科书上模板方法的定义有差异；因为sort要适用于所有数组，所以提供了一个Arrays工具类。但仍然是模板方法模式

### InputStream.read()

```
 //算法框架  
 public int read(byte b[], int off, int len) throws IOException {  
        ...  
  
        int c = read();  
          
        ...  
}  
  
//算法步骤由子类实现  
public abstract int read() throws IOException;  
```

### JFrame.paint()

``` 
// JFrame  
    public void update(Graphics g) {  
        paint(g);  
    }  

public class MyFrame extends JFrame {  
    public MyFrame(){  
        super();  
        this.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);  
        this.setSize(300,300);  
        this.setVisible(true);  
    }  
      
    @Override  
    public void paint(Graphics g){ //重定义算法步骤  
        super.paint(g);  
        g.drawString("I rule !", 100, 100);  
    }  
      
    public static void main(String[] args){  
        MyFrame frame = new MyFrame();  
    }  
  
} 
``` 



### Applet.init()/start()/stop()/destroy()/paint()

```
// Applet  
    public void init() { //什么也不做的hook  
    }  

// Beans  
   public static Object instantiate(ClassLoader cls, String beanName,   
                    BeanContext beanContext, AppletInitializer initializer)  
                        throws IOException, ClassNotFoundException {  
  
          
  
                // If it was deserialized then it was already init-ed.  
                // Otherwise we need to initialize it.  
  
                if (!serialized) {  
                    // We need to set a reasonable initial size, as many  
                    // applets are unhappy if they are started without  
                    // having been explicitly sized.  
                    applet.setSize(100,100);  
                    applet.init(); //调用hook  
                }  
  
                  
        }  
  
        return result;  
    }  
```    

Applet中的init()/start()/stop()/destroy()/paint()这些方法，都是hook。





## 2. Iterator

### Collection.iterator()

 
## 3. Adapter

### RunnableAdapter

完整类名：`java.util.concurrent.Executors.RunnableAdapter<T>`
 
我们知道`FutureTask`接受一个`Callable`参数，那如果我们现有的是`Runnable`该怎么办呢？
`FutureTask`本身提供了适配：

```
    /**
     * Creates a <tt>FutureTask</tt> that will upon running, execute the given <tt>Callable</tt>.
     */
    public FutureTask(Callable<V> callable) {
        sync = new Sync(callable);
    }
 
    /**
     * Creates a <tt>FutureTask</tt> that will upon running, execute the given <tt>Runnable</tt>
     */
    public FutureTask(Runnable runnable, V result) {
        sync = new Sync(Executors.callable(runnable, result));
    }
```
 
 Executors.callable()返回Adapter对象：

```
    public static <T> Callable<T> callable(Runnable task, T result) {
        return new RunnableAdapter<T>(task, result);
    }
 
    /** --Adapter!--
     * A callable that runs given task and returns given result
     */
    static final class RunnableAdapter<T> implements Callable<T> {  //Target
        final Runnable task; //Adaptee
        final T result;
        RunnableAdapter(Runnable  task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

AbstractExecutorService.submit()也用到了这个Adapter：

``` 
   public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```
 


未完待续。。。 
 
 