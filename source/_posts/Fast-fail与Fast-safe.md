---
title: Fast-fail与Fast-safe
date: 2019-08-01 20:12:50
categories: java
description: Fast-fail与Fast-safe
---



## Fast-fail

#### 是什么

**fail-fast,即快速失败机制，它是java集合中的一种错误检测机制，当多个线程或者单个线程,在结构上对集合进行改变时，就有可能会产生fail-fast机制。**

*注意：结构上的改变的意思是，例如集合上的**插入和删除**就是结构上的改变，但是，如果是对集合中某个元素进行**修改**的话，并不是结构上的改变。* 



#### 为什么

为什么要有快速失败？为什么用for(int i=0;i<list.size();i++)这种形式就能一边遍历一边修改集合？

其实答案很简单，普通for循环时，当一个元素结结构被改变时，集合大小和下标也会随之变化，**size是可以随着你改变变换的**。

但使用了迭代器就不行了。迭代器（Iterator）是工作在一个独立的线程中，并且拥有一个 mutex 锁。 迭代器被创建之后会建立一个指向原来对象的单链索引表，当原来的对象数量发生变化时，**这个索引表的内容不会同步改变**，所以当索引指针往后移动的时候就找不到要迭代的对象，所以按照 fail-fast 原则 迭代器会马上抛出 java.util.ConcurrentModificationException。

最后说一嘴，如果非要在用迭代器的时候删除，可以用迭代器的remove方法。如下

```java
  if (s.equals("a")) {
        iter.remove();
    }
```

#### 原理

迭代器在执行next()等方法的时候，都会调用**checkForComodification()**这个方法，查看modCount==expectedModCount如果不相等则抛出异常。

expectedModcount:这个值在对象被创建的时候就被赋予了一个固定的值modCount。也就是说这个值是不变的。也就是说，如果在迭代器遍历元素的时候，如果modCount这个值发生了改变，那么再次遍历时就会抛出异常。 

什么时候modCount会发生改变呢？就是对集合的元素的个数做出改变的时候，modCount的值就会被改变，如果删除，插入。但修改则不会。



```java
private class Itr implements Iterator<E> {
  
        int expectedModCount = modCount;
        
         public E next() {
            checkForComodification();
            ...省略...
       }
            final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
}
```



## Fast-safe

### 是什么

采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是**先复制原有集合内容**，在拷贝的集合上进行遍历。主要是为了在多线程下可以并发的对集合进行更改。

### 原理

由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。

###  缺点

基于拷贝内容的优点是避免了Concurrent Modification Exception，但同样地，**迭代器并不能访问到修改后的内容**，即迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。

## 两者区别

Fast-fail场景：java.util包下的集合类都是快速失败的，不能在多线程下发生并发修改（迭代过程中被修改）。

Fast-fail场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。

> [fail_fast和fail_safe的介绍及区别](https://blog.csdn.net/mlym521/article/details/82465126)

