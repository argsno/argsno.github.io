---
title: "Java HashMap 源码解析"
date: 2019-03-25T19:28:11+08:00
draft: true
---

> 基于OpenJDK 11



## 签名（signature）

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

可以看到`HashMap`继承了

- 标记接口[Cloneable](http://docs.oracle.com/javase/7/docs/api/index.html?java/lang/Cloneable.html)，用于表明`HashMap`对象会重写`java.lang.Object#clone()`方法，HashMap实现的是浅拷贝（shallow copy）。
- 标记接口[Serializable](http://docs.oracle.com/javase/7/docs/api/index.html?java/io/Serializable.html)，用于表明`HashMap`对象可以被序列化

比较有意思的是，`HashMap`同时继承了抽象类`AbstractMap`与接口`Map`，因为抽象类`AbstractMap`的签名为

```java
public abstract class AbstractMap<K,V> implements Map<K,V>
```

[Stack Overfloooow](http://stackoverflow.com/questions/14062286/java-why-does-weakhashmap-implement-map-whereas-it-is-already-implemented-by-ab)上解释到：

> 在语法层面继承接口`Map`是多余的，这么做仅仅是为了让阅读代码的人明确知道`HashMap`是属于`Map`体系的，起到了文档的作用

`AbstractMap`相当于个辅助类，`Map`的一些操作这里面已经提供了默认实现，后面具体的子类如果没有特殊行为，可直接使用`AbstractMap`提供的实现。

### [Cloneable](http://docs.oracle.com/javase/7/docs/api/index.html?java/lang/Cloneable.html)接口

```
<code>It's evil, don't use it. </code>
```

`Cloneable`这个接口设计的非常不好，最致命的一点是它里面竟然没有`clone`方法，也就是说我们自己写的类完全可以实现这个接口的同时不重写`clone`方法。

关于`Cloneable`的不足，大家可以去看看《[Effective Java](http://www.amazon.com/gp/product/B000WJOUPA/ref=as_li_qf_sp_asin_il_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B000WJOUPA&linkCode=as2&tag=job0ae-20)》一书的作者[给出的理由](http://www.artima.com/intv/bloch13.html)，在所给链接的文章里，Josh Bloch也会讲如何实现深拷贝比较好，我这里就不在赘述了。

### [Map](http://docs.oracle.com/javase/7/docs/api/index.html?java/util/Map.html)接口

在[Eclipse](http://res.importnew.com/eclipse)中的outline面板可以看到`Map`接口里面包含以下成员方法与内部类：
![img](http://ww3.sinaimg.cn/mw690/b254dc71jw1evxje5k5puj20h80gcgov.jpg)

Map_field_method可以看到，这里的成员方法不外乎是“增删改查”，这也反映了我们编写程序时，一定是以“数据”为导向的。

`Map`虽然并不是`Collection`，但是它提供了三种“集合视角”（collection views），与下面三个方法一一对应：

- `Set<K> keySet()`，提供key的集合视角
- `Collection<V> values()`，提供value的集合视角
- `Set<Map.Entry<K, V>> entrySet()`，提供key-value序对的集合视角，这里用内部类`Map.Entry`表示序对

### [AbstractMap](http://docs.oracle.com/javase/7/docs/api/index.html?java/util/AbstractMap.html)抽象类

`AbstractMap`对`Map`中的方法提供了一个基本实现，减少了实现`Map`接口的工作量。

举例来说：

> 如果要实现个不可变（unmodifiable）的map，那么只需继承`AbstractMap`，然后实现其`entrySet`方法，这个方法返回的set不支持add与remove，同时这个set的迭代器（iterator）不支持remove操作即可。
>
> 相反，如果要实现个可变（modifiable）的map，首先继承`AbstractMap`，然后重写（override）`AbstractMap`的put方法，同时实现`entrySet`所返回set的迭代器的remove方法即可。

## 设计理念（design concept）

### 哈希表（hash table）

`HashMap`是一种基于[哈希表（hash table）](https://en.wikipedia.org/wiki/Hash_table)实现的map，哈希表（也叫关联数组）一种通用的数据结构，大多数的现代语言都原生支持，其概念也比较简单：`key经过hash函数作用后得到一个槽（buckets或slots）的索引（index），槽中保存着我们想要获取的值`，如下图所示

![img](http://ww2.sinaimg.cn/mw690/b254dc71jw1evxje6n0o5j20gs0c8abh.jpg)

hash table demo很容易想到，一些不同的key经过同一hash函数后可能产生相同的索引，也就是产生了冲突，这是在所难免的。
所以利用哈希表这种数据结构实现具体类时，需要：

- 设计个好的hash函数，使冲突尽可能的减少
- 其次是需要解决发生冲突后如何处理。

后面会重点介绍`HashMap`是如何解决这两个问题的。

### HashMap的一些特点

- 线程非安全，并且允许key与value都为null值，`HashTable`与之相反，为线程安全，key与value都不允许null值。
- 不保证其内部元素的顺序，而且随着时间的推移，同一元素的位置也可能改变（resize的情况）
- put、get操作的时间复杂度为O(1)。
- 遍历其集合视角的时间复杂度与其容量（capacity，槽的个数）和现有元素的大小（entry的个数）成正比，所以如果遍历的性能要求很高，不要把capactiy设置的过高或把平衡因子（load factor，当entry数大于capacity*loadFactor时，会进行resize，reside会导致key进行rehash）设置的过低。
- 由于HashMap是线程非安全的，这也就是意味着如果多个线程同时对一hashmap的集合试图做迭代时有结构的上改变（添加、删除entry，只改变entry的value的值不算结构改变），那么会报[ConcurrentModificationException](http://docs.oracle.com/javase/7/docs/api/java/util/ConcurrentModificationException.html)，专业术语叫`fail-fast`，尽早报错对于多线程程序来说是很有必要的。
- `Map m = Collections.synchronizedMap(new HashMap(...));` 通过这种方式可以得到一个线程安全的map。

## 源码剖析

首先从构造函数开始讲，`HashMap`遵循集合框架的约束，提供了一个参数为空的构造函数与有一个参数且参数类型为Map的构造函数。除此之外，还提供了两个构造函数，用于设置`HashMap`的容量（capacity）与平衡因子（loadFactor）。

 ```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
 ```

从代码上可以看到，容量与平衡因子都有个默认值，并且容量有个最大值

 ```java
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
 ```

可以看到，默认的平衡因子为0.75，这是权衡了时间复杂度与空间复杂度之后的最好取值（JDK说是最好的），过高的因子会降低存储空间但是查找（lookup，包括HashMap中的put与get方法）的时间就会增加。

这里比较奇怪的是问题：容量必须为2的指数倍（默认为16），这是为什么呢？解答这个问题，需要了解HashMap中哈希函数的设计原理。

### 哈希函数的设计原理

```java
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

### HashMap.Entry

HashMap中存放的是HashMap.Node对象，它继承自Map.Entry，其比较重要的是构造函数

```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

