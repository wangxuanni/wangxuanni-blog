---
title: java多线程
date: 2019-02-11 23:53:50
categories: java
description: 多线程基础概念
---
# 两个重要的修饰符
## synchronized：“这条桥上一次只能过一个人”
synchronized保证了当有**多个线程同时操作共享数据时，任何时刻只有一个线程能进入临界区操作共享数据，其他线程必须等待。**
它可以保证操作的**原子性**synchronized通过**同步锁**保证线程安全，进入临界区前必须获得对象的锁，其他没有获得锁的线程不可进入。当临界区中的线程操作完毕后，它会释放锁，此时其他线程可以竞争锁，得到锁的那个线程便可以进入临界区。
synchronized还可以保证**可见性**。因为对一个变量的unlock操作之前，必须先把次变量同步回主内存中。它还可以保证有序性，因为一个变量在任何时刻只能有一个线程对其进行lock操作（也就是任何时刻只有一个线程可以获得该锁对象），这决定了持有同一把锁的两个同步块只能串行进入。

## volatile
volatile是一个关键字，用于修饰变量。被其修饰的变量具有可见性和有序性。
**可见性**，当一条线程修改了这个变量的值，新值能被其他线程立刻观察到。具体来说，volatile的作用是：在本CPU对变量的修改直接写入主内存中，同时这个写操作使得其他CPU中对应变量的缓存行无效，这样其他线程在读取这个变量时候必须从主内存中读取，所以读取到的是最新的，这就是上面说得能被立即“看到”。
**有序性**，即volatile可以**禁止指令重排**。volatile在其汇编代码中有一个lock操作，这个操作相当于一个内存屏障，指令重排不能越过内存屏障。具体来说在执行到volatile变量时，内存屏障之前的语句一定被执行过了且结果对后面是已知的，而内存屏障后面的语句一定还没执行到；在volatile变量之前的语句不能被重排后其之后，相反其后的语句也不能被重排到之前。


## 线程安全

线程安全简单来说就在多线程的情况下也不会有问题，看似一句废话，要怎么理解呢
比如ArrayList不是线程安全的就是一个线程不安全的类，
如果两个线程对可以同一个ArrayList进行add操作会出现什么结果？请看下面代码
```
public class MyTest {
            static List<Integer> list = new ArrayList<>();

            static class BB implements Runnable {
                @Override
                public void run() {
                    for (int j = 0; j < 10000; j++) {
                        list.add(j);
                    } } }

public static void main(String[] args) throws InterruptedException{

                BB b = new BB();
                Thread t1 = new Thread(b);
                Thread t2 = new Thread(b);
                t1.start();
                t2.start();
                t1.join();
                t2.join();

                System.out.println(list.size());
            }
        }
```
问题出在add方法
```
public boolean add(E e) {   ensureCapacityInternal(size + 1);  // Increments modCount!!   
elementData[size++] = e;  
return true;}
```
上面的程序，可能有三种情况发生：

* 数组下标越界。首先要检查容量，必要时进行扩容。每当在数组边界处，如果A线程和B线程同时进入并检查容量，也就是它们都执行完ensureCapacityInternal方法，因为还有一个空间，所以不进行扩容，此时如果A暂停下来，B成功自增；然后接着A从 elementData[size++]=e开始执行，由于A之前已经检查过没有扩容，而B成功自增使得现在没有空余空间了，此时A就会发生数组下标越界。

* 小于20000。size++可以看成是 size=size+1，这一行代码包括三个步骤，先读取size，然后将size加1，最后将这个新值写回到size。此时若A和B线程同时读取到size假设为10，B先自增成功size变11，然后回来A因为它读到的size也是10，所以自增后写入size被更新成11，也就是说两次自增，实际上size只增大了1。因此最后的size会小于200。
* 等于20000 很幸运，没有发生上面情况
顺便说一句，线程越多，或者加的数越大越可能出现不安全的问题

# 生产者消费者
一个非常典型性的线程交互的问题。

1. 使用栈来存放数据
1.1 把栈改造为支持线程安全
1.2 把栈的边界操作进行处理，当栈里的数据是0的时候，访问pull的线程就会等待。 当栈里的数据是200的时候，访问push的线程就会等待
2. 提供一个生产者（Producer）线程类，生产随机大写字符压入到堆栈3. 提供一个消费者（Consumer）线程类，从堆栈中弹出字符并打印到控制台




