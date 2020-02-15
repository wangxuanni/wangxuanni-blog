---
title: 用java实现一个LRU
date: 2019-07-25 16:36:50
categories: 算法
description: Least Recently Used(LRU)，即最近最少使用页面置换算法。
---

# 概念

Least Recently Used(LRU)，即最近最少使用页面置换算法。**选择最长时间没有被引用的页面进行置换**，思想是：如果一个页面很久没有被引用到，那么可以认为在将来该页面也很少被访问到。

在操作系统中的应用：当发生缺页（CPU要访问的页不在内存中），计算内存中每个页上一次被访问的时间，置换上次使用到当前时间最长的一个页面。

使用java可以有两种实现方式。

# 第一种：双向链表+哈希表

可以使用**双向链表+哈希表**的方式。**HashMap主要是为了判断是否命中缓存。LinkedList用于维护一个按最近一次访问时间排序的页面链表。**链表头结点是最近刚刚访问过的页面，链表尾结点是最久未被访问的页面。访问内存时，若命中缓存，找到响应的页面，将其移动到链表头部，表示该页面是最近刚刚访问的。缺页时，将链表尾部的页面移除，同时新页面放到链表头。

该类有四个方法：

- moveToFirst()：把该元素移动链表的头部
- removeLast()：把链表尾部元素删除。
- get()：同步获取元素,map未命中，返回null。map命中则获取，并调用moveToFirst。
- put()：同步放入元素。如果map未命中，如果链表长度已经超过缓存的大小，移除链表尾部的元素，把元素放入链表的头部和map里。如果map命中，则moveToFirst把该元素移到链表的头。



```java
public class LRUCache<K, V> {
    public static void main(String[] args) {
        LRUCache<String,String> lru=new LRUCache<>(10);
        lru.put("C", null);
        lru.put("A", null);
        lru.put("D", null);
        lru.put("B", null);
        lru.put("E", null);
        lru.put("B", null);
        lru.put("A", null);
        lru.put("B", null);
        lru.put("C", null);
        lru.put("D", null);

        System.out.println(lru);
        /*
        out:[D, C, B, A, E]
        */

    }
    //缓存大小，put时候需判断有没超过缓存大小
    private final int cacheSize;
    //用散列表判断是否命中缓存。
    private HashMap<K, V> map = new HashMap<>();
    //用链表维护最近一次访问时间排序的页面链表
    private LinkedList<K> linkCache = new LinkedList();

    LRUCache(int cacheSize) {
        this.cacheSize = cacheSize;
    }
    
    private synchronized void removeLast() {
        linkCache.removeLast();
    }
    
//用syn保证线程安全
    public synchronized void put(K key, V val) {
        if (!map.containsKey(key)) {
            if (map.size() >= cacheSize) {
                removeLast();
            }
            map.put(key, val);
            linkCache.addFirst(key);//put加到链表头
        } else {
            moveToFirst(key);
        }
    }

    public synchronized V get(K key) {
        if (!map.containsKey(key)) {
            return null;
        }
        moveToFirst(key);
        return map.get(key);
    }

    private synchronized void moveToFirst(K key) {
        linkCache.remove(key);
        linkCache.addFirst(key);

    }
    
    @Override
    public String toString() {
        return linkCache.toString();
    }
}

```



# 第二种：LinkedHashMap

JDK其实已经实现了一种顺序存储的HashMap，我们可以直接使用。代码如下



```java
package other;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * @description:
 * @author: wangxuanni
 * @create: 2019-08-13 23:40
 **/

public class LRU3<K, V> {
    public static void main(String[] args) {
        LRU3<String, Integer> lru3 = new LRU3<>(3);
        lru3.put("A", 1);
        lru3.put("B", 2);
        lru3.put("C", 3);
        lru3.put("D", 4);
        lru3.put("A", 1);
        lru3.put("B", 2);
        lru3.print();
        //out：D--A--B--

    }

    private static final float hashLoadFactory = 0.75f;
    private LinkedHashMap<K, V> map;
    private int cacheSize;

    public LRU3(int cacheSize) {
        this.cacheSize = cacheSize;
        int capacity = (int) Math.ceil(cacheSize / hashLoadFactory) + 1;
        // 最后一个参数true 表示按访问顺序来进行排序
        map = new LinkedHashMap<K, V>(capacity, hashLoadFactory, true) {
            private static final long serialVersionUID = 1;
       // 当 map中的数据量大于cacheSize的时候，就自动删除最老的数据。
            @Override
            protected boolean removeEldestEntry(Map.Entry eldest){
                return size() > LRU3.this.cacheSize;
            }
        };
    }

    public synchronized void put(K key, V value) {
        map.put(key, value);
    }

    public void print() {
        for (Map.Entry<K, V> entry : map.entrySet()) {
            System.out.print(entry.getKey() + "--");
        }
        System.out.println();
    }
}

```



## LinkedHashMap原理

LinkedHashMap继承了HashMap，作为HashMap的亲儿子，它们有很多相似的地方。

### 构造方法

```java
    public LinkedHashMap() {
        // 调用HashMap的构造方法，其实就是初始化Entry[] table
        super();
        // 这里是指是否基于访问排序，默认为false
        accessOrder = false;
    }
```

这里有一个重要的成员变量accessOrder。因LinkedHashMap存储数据是有序的，而且分为两种：插入顺序和访问顺序。accessOrder则表示是否按访问顺序存储。这里设置为**false**，表示是**插入顺序存储的**，这也是默认值。

在上面代码中，构造函数的最后一个参数就是把accessOrder设置为**true**，表示按访问顺序排序

```java
map = new LinkedHashMap<K, V>(capacity, hashLoadFactory, true)
```

### Entry

LinkedHashMap与HashMap最大的不同在于它的Entry有**前后指针—**— Entry<K,V> before, after

```cpp
    /**
     * LinkedHashMap entry.
     */
    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }
```

此外，在初始化的时候除了初始化了一个Entry[] table，还会额外初始化一个hash=-1，key、value、next都为null的**只有头结点的双向链表**，如图

![img](https://upload-images.jianshu.io/upload_images/4843132-cac0b65e4d23b6bd.png?imageMogr2/auto-orient/strip|imageView2/2/w/524/format/webp)

```java
@Override
void init() {
        // 创建了一个hash=-1，key、value、next都为null的Entry
        header = new Entry<>(-1, null, null, null);
        // 让创建的Entry的before和after都指向自身，注意after不是之前提到的next
        // 其实就是创建了一个只有头部节点的双向链表
        header.before = header.after = header;
    }
```

### put元素

当put元素时，不但要把它加入到HashMap中去，还要加入到双向链表中。

首先是只加入一个元素Entry1，假设index为0：

![img](https://upload-images.jianshu.io/upload_images/4843132-d48bb58775418c95.png?imageMogr2/auto-orient/strip|imageView2/2/w/475/format/webp)

LinkedHashMap结构一个元素.png

当再加入一个元素Entry2，假设index为15：

![img](https://upload-images.jianshu.io/upload_images/4843132-10c917166c1f4745.png?imageMogr2/auto-orient/strip|imageView2/2/w/525/format/webp)

LinkedHashMap结构两个元素.png

### 双向链表的重排序

当key如果已经存在时，并且accessOrder为true，即是访问顺序模式，才会put时对更新的Entry进行重新排序，而如果是插入顺序模式时，不会重新排序，这里的排序跟在HashMap中存储没有关系，只是指在双向链表中的顺序。

举个栗子：开始时，HashMap中有Entry1、Entry2、Entry3，并设置LinkedHashMap为访问顺序，则更新Entry1时，会先把Entry1从双向链表中删除，然后再把Entry1加入到双向链表的表尾，而Entry1在HashMap结构中的存储位置没有变化。





最后，有一个面试问题是如何实现顺序存储的hashmap，看完这篇文章，可以轻松回答了：可以使用双向链表加哈希表，也可以参照LinkedHashMap实现原理。

> [图解LinkedHashMap原理，关于LinkedHashMap大部分原理都来自该博文](https://www.jianshu.com/p/8f4f58b4b8ab)