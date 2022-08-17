---
layout: post
title: "How to Implement a HashMap "
date: 2014-11-17 19:24:35 +0800
comments: true
categories: [Java]
tags: [Java, Source Code, JDK, Collection]  
keywords: 分析HashMap源码及实现原理，研究如何实现HashMap

---

作为一个Java开发民工，对`HashMap`可以说是经常使用的，不过要是想更好地使用它，还是需要了解其内部实现原理的。当了解了实现原理后，自己实现一个HashMap也就不难了。

<!--more-->

## 1. HashMap的数据结构

首先思考一下`HashMap`底层可以用什么数据结构来存储数据呢？可以用数组或者链表，这两者基本上是两个极端。

- 数组

数组的存储区间是连续的，占用内存严重，故空间复杂度很大；不过数组的二分查找时间复杂度小，为O(1)。

数组的特点是：寻址容易，插入和删除困难。

- 链表

链表存储区间离散，占用内存比较宽松，故空间复杂度很小；但时间复杂度很大，达O(N)。

链表的特点是：寻址困难，插入和删除容易。

- 哈希表

那么我们能不能综合两者的特性，做出一种寻址容易，插入删除也容易的数据结构？答案是肯定的，这就是我们要提起的哈希表。  
哈希表（(Hash table）既满足了数据的查找方便，同时不占用太多的内容空间，使用也十分方便。  

哈希表有多种不同的实现方法，最常用的一种方法被称为—— `拉链法`，我们可以理解为“链表的数组” ，如图：
![map](/images/post/2014/11/hashmap.png)  
    

从上图我们可以发现哈希表是由`数组 + 链表`组成的，数组中每个元素存储的是一个链表的头结点。  
那么这些元素是按照什么样的规则存储到数组中呢。一般情况是通过hash(key)%len获得，也就是元素的key的哈希值对数组长度取模得到。例如，12%16=12,28%16=12,108%16=12,140%16=12。所以12、28、108以及140都存储在数组下标为12的位置。  

HashMap其实也是一个线性的数组实现的,所以可以理解为其存储数据的容器就是一个线性数组。这可能让我们很不解，一个线性的数组怎么实现按键值对来存取数据呢？这里HashMap有做一些处理。

首先HashMap里面实现一个静态内部类`Entry`，其重要的属性有 key, value, next，从属性key, value我们就能很明显的看出来Entry就是HashMap键值对实现的一个基础bean，我们上面说到HashMap的基础就是一个线性数组，这个数组就是`Entry[]`，Map里面的内容都保存在Entry[]里面。  

```  
    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    transient Entry[] table;
```  
    
## 2. HashMap的存取实现

既然是线性数组，如何能随机存取？这里HashMap用了一个小算法，大致是这样实现：  
```
// 存储时:
int hash = key.hashCode(); // 这个hashCode方法这里不详述,只要理解每个key的hash是一个固定的int值
int index = hash % Entry[].length;
Entry[index] = value;
```  

``` 
// 取值时:
int hash = key.hashCode();
int index = hash % Entry[].length;
return Entry[index];
``` 
 
### 2.1 put

 
> 先思考一个问题：如果两个key通过hash%Entry[].length得到的index相同，会不会有覆盖的危险？  

答案是HashMap里面用到链式数据结构的一个概念。上面我们提到过Entry类里面有一个next属性，作用是指向下一个Entry。打个比方， 第一个键值对A进来，通过计算其key的hash得到的index = 0，记做：Entry[0] = A。一会后又进来一个键值对B，通过计算其index也等于0，现在怎么办？HashMap会这样做: B.next = A,Entry[0] = B；如果又进来C，index也等于0，那么C.next = B, Entry[0] = C；这样我们发现index=0的地方其实存取了A,B,C三个键值对，他们通过next这个属性链接在一起。**也就是说数组中存储的是最后插入的元素。** 

这里其实是一种解决hash冲突的方法，一般可以用如下方法来解决hash冲突：  
- 开放定址法（线性探测再散列，二次探测再散列，伪随机探测再散列）  
- 再哈希法  
- 链地址法  
- 建立一个公共溢出区  
HashMap的解决办法就是采用`链地址法`。


看看源码是如何实现的：  

```
 public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value); //null总是放在数组的第一个链表中
        int hash = hash(key.hashCode());
        int i = indexFor(hash, table.length);
        //遍历链表
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            //如果key在链表中已存在，则替换为新value
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
    
 
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<K,V>(hash, key, value, e); //参数e, 是Entry.next
    //如果size超过threshold，则扩充table大小。再散列
    if (size++ >= threshold)
            resize(2 * table.length);
}


static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
    
``` 

HashMap里面也包含一些优化方面的实现。比如：Entry[]的长度一定后，随着map里面数据的越来越长，这样同一个index的链就会很长，会不会影响性能？HashMap里面设置一个因子，随着map的size越来越大，Entry[]会以一定的规则加长长度。

### 2.2 get

```
 public V get(Object key) {
        if (key == null)
            return getForNullKey();
        int hash = hash(key.hashCode());
        //先定位到数组元素，再遍历该元素处的链表
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
}
```

 
### 2.3 null key的存取

null key总是存放在Entry[]数组的第一个元素。  

```
   private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);
        return null;
    }
 
    private V getForNullKey() {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null)
                return e.value;
        }
        return null;
    }  
    
```   
 
 
 
### 2.4 确定数组index：hashcode % table.length取模

HashMap存取时，都需要计算当前key应该对应Entry[]数组哪个元素，即计算数组下标；算法如下：

```
   /**
     * Returns index for hash code h.
     */
    static int indexFor(int h, int length) {
        return h & (length-1);
    }
```
 
按位取并，作用上相当于取模mod或者取余%。  

key经过hash后，可以取模来进行放入数组，也不会出现越界的情况，
之所以没有使用取模，而是按位与的形式，是因为计算机的二进制运算效率比取模效率高。
 
> 这意味着：数组下标相同，并不表示hashCode相同。
 

### 2.5 table初始大小


``` 
  public HashMap(int initialCapacity, float loadFactor) {
        .....
        // Find a power of 2 >= initialCapacity
        int capacity = 1;
        while (capacity < initialCapacity)
            capacity <<= 1;
        this.loadFactor = loadFactor;
        threshold = (int)(capacity * loadFactor);
        table = new Entry[capacity];
        init();
    }
```
 
注意table初始大小并不是构造函数中的initialCapacity！！
而是 >= initialCapacity的2的n次幂！！！！

> 为什么这么设计呢？为什么HashMap的大小要是2的指数次呢？

 
 
回顾一下数字索引的算法是`h & (length-1)`。如果Map的大小不是2的指数次，我们设置为7，7的二进制是：111，（length-1）大小是6，6的二进制是：110
结果如下，有些数组中的位置没有被设置，有些重复了，一是导致空间浪费，同时增加了碰撞的几率。

```
for(int i=0;i<10;i++){
    System.out.println
    ("数值i="+i+", 二进制="+Integer.toBinaryString(i)+"（"+Integer.toBinaryString(6)+"）"+" ,和6按位与="+(i&6));
}
   
数值i=0, 二进制=0（110） ,和6按位与=0
数值i=1, 二进制=1（110） ,和6按位与=0
数值i=2, 二进制=10（110） ,和6按位与=2
数值i=3, 二进制=11（110） ,和6按位与=2
数值i=4, 二进制=100（110） ,和6按位与=4
数值i=5, 二进制=101（110） ,和6按位与=4
数值i=6, 二进制=110（110） ,和6按位与=6
数值i=7, 二进制=111（110） ,和6按位与=6
数值i=8, 二进制=1000（110） ,和6按位与=0
数值i=9, 二进制=1001（110） ,和6按位与=0
```

然后我们设置8，（length-1）大小是7，7的二进制是111，打印看结果，空间充分利用，并且减少了碰撞的几率。

```
for(int i=0;i<10;i++){
      System.out.println(
        ​"数值i="+i+", 二进制="+Integer.toBinaryString(i)+"（"+Integer.toBinaryString(7)+"）"+" ,和7按位与="+(i&7));
}
​
数值i=0, 二进制=0（111） ,和7按位与=0
数值i=1, 二进制=1（111） ,和7按位与=1
数值i=2, 二进制=10（111） ,和7按位与=2
数值i=3, 二进制=11（111） ,和7按位与=3
数值i=4, 二进制=100（111） ,和7按位与=4
数值i=5, 二进制=101（111） ,和7按位与=5
数值i=6, 二进制=110（111） ,和7按位与=6
数值i=7, 二进制=111（111） ,和7按位与=7
数值i=8, 二进制=1000（111） ,和7按位与=0
数值i=9, 二进制=1001（111） ,和7按位与=1
```
 
## 3. 再散列rehash过程

当哈希表的容量超过默认容量时，必须调整table的大小。当容量已经达到最大可能值时，那么该方法就将容量调整到Integer.MAX_VALUE返回，这时，需要创建一张新表，将原表的映射到新表中。

```
   /**
     * Rehashes the contents of this map into a new array with a
     * larger capacity.  
     * This method is called automatically when the
     * number of keys in this map reaches its threshold.
     *
     *
     * @param newCapacity the new capacity, MUST be a power of two;
     *        must be greater than current capacity unless current
     *        capacity is MAXIMUM_CAPACITY (in which case value
     *        is irrelevant).
     */
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable);
        table = newTable;
        threshold = (int)(newCapacity * loadFactor); //设置阈值
    }
``` 
注意设置threshold时用到了`loadFactor`，它是在构造HashMap时指定的，默认值是 `0.75f`。

```    
 
    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable) {
        Entry[] src = table;
        int newCapacity = newTable.length;
        for (int j = 0; j < src.length; j++) {
            Entry<K,V> e = src[j];
            if (e != null) {
                src[j] = null;
                do {
                    Entry<K,V> next = e.next;
                    //重新计算index
                    int i = indexFor(e.hash, newCapacity);
                    e.next = newTable[i];
                    newTable[i] = e;
                    e = next;
                } while (e != null);
            }
        }
    }
    
``` 

> rehash过程是要消耗资源的，应该尽量避免rehash.


    
## 4. 实现自己的HashMap

理解和JDK对HashMap的实现原理后，可以自己动手模仿一个：
 
Entry.java


```  
public class Entry<K,V>{
    final K key;
    V value;
    Entry<K,V> next;//下一个结点
 
    //构造函数
    public Entry(K k, V v, Entry<K,V> n) {
        key = k;
        value = v;
        next = n;
    }

    public final K getKey() {
        return key;
    }

    public final V getValue() {
        return value;
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (!(o instanceof Entry))
            return false;
        Entry e = (Entry)o;
        Object k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            Object v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public final int hashCode() {
        return (key==null   ? 0 : key.hashCode()) ^ (value==null ? 0 : value.hashCode());
    }

    public final String toString() {
        return getKey() + "=" + getValue();
    }

}
``` 

MyHashMap.java



``` 
//保证key与value不为空
public class MyHashMap<K, V> {
    private Entry[] table;//Entry数组表
    static final int DEFAULT_INITIAL_CAPACITY = 16;//默认数组长度
    private int size;

    // 构造函数
    public MyHashMap() {
        table = new Entry[DEFAULT_INITIAL_CAPACITY];
        size = DEFAULT_INITIAL_CAPACITY;
    }

    //获取数组长度
    public int getSize() {
        return size;
    }
    
    // 求index
    static int indexFor(int h, int length) {
        return h % (length - 1);
    }

    //获取元素
    public V get(Object key) {
        if (key == null)
            return null;
       int hash = key.hashCode();// key的哈希值
        int index = indexFor(hash, table.length);// 求key在数组中的下标
        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            Object k = e.key;
            if (e.key.hashCode() == hash && (k == key || key.equals(k)))
                return e.value;
        }
        return null;
    }

    // 添加元素
    public V put(K key, V value) {
        if (key == null)
            return null;
        int hash = key.hashCode();
        int index = indexFor(hash, table.length);

        // 如果添加的key已经存在，那么只需要修改value值即可
        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            Object k = e.key;
            if (e.key.hashCode() == hash && (k == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                return oldValue;// 原来的value值
            }
        }
        // 如果key值不存在，那么需要添加
        Entry<K, V> e = table[index];// 获取当前数组中的e
        table[index] = new Entry<K, V>(key, value, e);// 新建一个Entry，并将其指向原先的e
        return null;
    }

}
```  

 