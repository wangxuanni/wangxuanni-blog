---
title: 限流算法及RateLimiter源码解析
date: 2020-04-17 21:29:50
categories: 并发
description: 超详细！限流算法及RateLimiter源码解析
---









## 前言：

之所以对RateLimiter感兴趣，是之前实习的时候有一个需求，需要调用一个qps为10的合成图片接口。因为达不到实时调用的要求，我就写了一个定时任务事先调用该接口把结果放在缓存里面，外部不调用该接口而是去缓存里查数据。重点来了，因为需要合成的图片大概有两三百张，肯定不能直接一次调用该接口，于是用了限流去慢慢调用。代码类似这样：

```java
  RateLimiter rateLimiter = RateLimiter.create(10);
        for (int i = 1; i < 200; i++) {
            rateLimiter.acquire();
            System.out.println("生成第"+i+"张合成图片");
        }
```

当时只是简单的使用了，并没有深究其原理，现在放假有空就把他的源码撸了一遍。



## 令牌桶算法：



在学习令牌桶算法之前我们现在来看看几种比较简单的限流算法，了解它们有什么问题，令牌桶算法是怎么解决这些问题的

### 计数器算法

计数器算法是限流算法里最简单也是最容易实现的一种算法。每个时间段，比如1min，都有一个计数器，到了下一分钟则重置计数器。

缺陷具有**边界问题**。用户通过在时间窗口的重置节点处突发请求， 可以瞬间超过我们的速率限制，比如0:59时发送了100个请求，并且1:00又发送了100个请求，也就是1秒内发送了200个请求。这个计数器算法的漏洞，可能导致应用崩溃。

那么如何将临界问题的影响降低呢？我们可以看下面的滑动窗口算法。

**滑动窗口**

![](https://wangxuanni.oss-cn-hongkong.aliyuncs.com/ratelimiter/%E6%BB%91%E5%8A%A8.jpg)

 滑动窗口会将一分钟进行划分，比如图中，我们就将1分钟划成了6格，所以每格代表的是10秒钟。每过10秒钟，我们的时间窗口就会往右滑动一格。每**一个格子都有自己独立的计数器counter**。

那么滑动窗口怎么解决刚才的临界问题的呢？我们可以看上图，0:59到达的100个请求会落在灰色的格子中，而1:00到达的请求会落在橘黄色的格 子中。当时间到达1:00时，我们的窗口会往右移动一格，那么此时时间窗口内的总请求数量一共是200个，超过了限定的100个，所以此时能够检测出来触 发了限流。

再来回顾一下刚才的计数器算法，**计数器算法其实就是滑动窗口算法。只是它没有对时间窗口做进一步地划分，只有1格。**由此可见，当滑动窗口的格子划分的越多，那么滑动窗口的滚动就越平滑，限流的统计就会越精确。

### 漏桶算法

漏桶算法顾名思义就是注水漏水过程，往桶中以一定速率流出水，以任意速率流入水，当水超过桶流量则丢弃，因为桶容量是不变的，保证了整体的速率。它是可以**限制瞬时并发数**。特点是

- 存下请求
- 匀速处理
- 多于丢弃



漏桶算法的实现往往依赖于队列，请求到达如果队列未满则直接放入队列，然后有一个处理器按照固定频率从队列头取出请求进行处理。如果请求量大，则会导致队列满，那么新来的请求就会被抛弃。

Nginx的限流模块就是基于漏桶算法的，它最大的特点就是**强行限制流量按照指定的比例下发**，适合那种对流量有绝对要求的场景，就是流量可以容许在我指定的值之下，可以被多次打回，但是无论如何决不能超过指定的。

![img](https://wangxuanni.oss-cn-hongkong.aliyuncs.com/ratelimiter/%E6%BC%8F%E6%A1%B6.png)

**缺陷：以均匀的速率，是无法应对短时间的突发流量。**



### 令牌桶算法

令牌桶算法整个的过程是这样的：

- 系统以恒定的速率产生令牌，然后将令牌放入令牌桶中
- 令牌桶有一个容量，当令牌桶满了的时候，再向其中放入的令牌就会被丢弃
- 每次一个请求过来，需要从令牌桶中获取一个令牌，假设有令牌，那么提供服务；假设没有令牌，那么拒绝服务

![img](https://wangxuanni.oss-cn-hongkong.aliyuncs.com/ratelimiter/%E6%A1%B6%E4%BB%A4%E7%89%8C.png)

它可以说是对漏桶算法的一种改进，它的优点是可接受突然大的流量。 桶算法能够限制请求调用的速率，而令牌桶算法能够在限制调用的平均速率的同时还允许一定程度的突发调用，即可以**限制时间窗口内的平均速率**。假设我们想要的速率是1000QPS，那么往桶中放令牌的速度就是1000个/s，假设第1秒只有800个请求，那意味着第2秒可以容许1200个请求，这就是**一定程度**突发流量的意思，反之我们看漏桶算法，第一秒只有800个请求，那么全部放过，第二秒这1200个请求将会被打回200个。



## RateLimiter源码解析：

### 介绍：



突发流量预支处理

### 关键变量

首先我们看下几个比较关键的变量：

- storedPermits 目前桶里令牌数
- maxPermits 最大的令牌保存量，即桶大小
- stableIntervalMicros 添加一个令牌到桶中的时间间隔
- long nextFreeTicketMicros = 0L  下次可获得令牌的时间，当一个请求被授权之后（通过acquire可以预定），这个时间会被继续往后推，大令牌量的请求会比少量的请求推的更远。

### create()

调用create接口时，实际实例化的为SmoothBursty类

```
public static RateLimiter create(double permitsPerSecond) {
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
}
 
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
}
```



### acquire()

```
public double acquire() {
    return acquire(1);
  }
  
  public double acquire(int permits) {
    long microsToWait = reserve(permits);
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    //把毫秒转为秒 SECONDS.toMicros(1L)=1000000
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
  }
```



第一个acquire无参方法委托到acquire(1)

第二个acquire()

调用reserve()方法得到获取permits个令牌需要的等待时间

通过stopwatch直接无中断地sleep这么长的时间

返回等待的时间毫秒数。

再点进去看看reserve方法做了什么：

```
  final long reserve(int permits) {
    checkPermits(permits);
    synchronized (mutex()) {
      return reserveAndGetWaitLength(permits, stopwatch.readMicros());
    }
  }
```

做一些参数检验

获取互斥锁

调用reserveAndGetWaitTime，传入需要获取的令牌数和当前的毫秒数。

再点进去看看reserveAndGetWaitLength方法做了什么：

```
final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
  }
```

这一段代码通过调用reserveEarliestAvailable来得到该请求能够获取令牌授权的毫秒时刻，然后通过运算返回得到需要等待的毫秒数

再点进去看看reserveEarliestAvailable方法：

```
  abstract long reserveEarliestAvailable(int permits, long nowMicros);
```

emmm，在RateLimiter类里它是一个抽象方法。点进它的实现发现是在SmoothRateLimiter类

```java
  @Override
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);

    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }

```

这个方法很关键，也比较复杂的，且听我逐行解析。

首先调用resync，更新令牌桶

先将下次能获得令牌的时间先存起来

判断桶里的令牌数够不够用，**storedPermitsToSpend可获取的令牌数，freshPermits为不够用的令牌数**，当然在够用的情况下storedPermitsToSpend就等于请求数而freshPermits等于0。

比如请求是10个，桶里有5个，套入上面代码则是：storedPermitsToSpend（5个） = min(requiredPermits（10个）, this.storedPermits（5个）

freshPermits（5个） = requiredPermits（10个） - storedPermitsToSpend（5个）

再比如请求是1个，桶里有5个，套入上面代码则是：storedPermitsToSpend（1个） = min(requiredPermits（1个）, this.storedPermits（5个）

freshPermits（0个） = requiredPermits（1个） - storedPermitsToSpend（0个）

然后算出需要等待的毫秒，这里用了一个抽象方法storedPermitsToWaitTime，它有两个实现SmoothWarmingUp和SmoothBursty。这主要是为了SmoothWarmingUp时用的，因为SmoothBursty的这个方法直接返回0。

下次可获得令牌的时间更新为：它本身+本次需要等待的时间。注意这里支持令牌预分发。

桶里的令牌数减去本次拿走的令牌数

返回直接保存好的nextFreeTicketMicros（下次能获得令牌的时间）

至此reserveEarliestAvailable方法就解析完了。



acquire()小朋友的调用链还真是有点长。总结一下，目前我们已经往下点进去了4个方法了。通过IDEA导航栏里的Navigate >>Call Hierarchy可以看查看它的调用链。

![0f224caa9cb48ffbe107cc2bda862fe](https://wangxuanni.oss-cn-hongkong.aliyuncs.com/ratelimiter/%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.png)



#### 令牌预分发

现在让我们来看看刚才更新下次可获得令牌的时间提到的令牌预分发是什么意思。

```java
this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
```

当限流器当前处于空闲状态时，一个大量令牌请求进来的时候，可以提前预授权给他足够的令牌让它能够立即执行，并推迟后续请求的等待时间（如之前所述），因此才会出现nowMicros < nextFreeTicketMicro的情况，而这种情况就说明当前仍处于对于之前一个请求的预授权阶段，不需要更新storedPermits，否则就还是nowMicros >= nextFreeTicketMicro的情况。

为什么需要令牌预分发呢？如果QPS=5的需求，限流算法需要保证没有请求能够在上个请求之后的200ms内获得授权。（至于为什么不是一秒发五个，而要把时间段分细，请往上看看计算器算法的问题。）

这次的问题在于，如果很长时间都没有请求呢？限流器处于一种**低利用率**的状态，然而它只记录新的请求的时间戳，下个请求也只能在这个请求之后的200ms之后才能获得授权，也就是QPS=2。这显然与我们期望的QPS不太匹配，并最终会导致低利用率或者请求溢出。**低利用率本应该意味着有多的资源可以立即使用。**比如说

- 第一次请求过来需要获取1个令牌，直接拿到
- RateLimiter在1秒钟后放一个令牌，第一次请求预支的1个令牌还上了
- 1秒钟之后第二次请求过来需要获得5个令牌，直接拿到
- RateLimiter在花了5秒钟放了5个令牌，还上了第二次请求预支的5个令牌
- 第三个请求在5秒钟之后拿到3个令牌

突发流量的处理，在令牌桶算法中有两种方式，一种是有足够的令牌才能消费，一种是先消费后还令牌。后者就像我们0首付买车似的，30万的车很少有等攒到30万才全款买的，先签了相关合同把车子给你，然后贷款慢慢还，这样就爽了。RateLimiter也是同样的道理，先让请求得到处理，再慢慢还上预支的令牌，客户端同样也爽了，否则我假设预支60个令牌，1分钟之后才能处理我的请求，不合理也不人性化。

因此我们需要添加另外一个衡量维度——storedPermits（桶里令牌数）。当storedPermits（桶里令牌数）为0，令牌都被用光了，代表没有低利用率存在。当storedPermits（桶里令牌数）=桶容量，令牌没有被使用，代表有低利用率存在。

RateLimiter允许某次请求拿走了超出剩余令牌数的令牌，但是下一次请求将为此付出代价，一直等到令牌亏空补上，并且桶中有足够本次请求使用的令牌为止。

此外它还构建了一个自定义注解，方便松耦合，灵活的对服务进行限流。



#### resync()令牌懒加载机制

如果是由你来写rateLImiter，可以理所当然想到需要一个定时插入令牌的方法。但Guava并没有这么做，而采用了触发式的更新令牌桶机制，即懒加载。**每次请求到来的时候才去执行令牌插入工作和其他字段如nextFreeTicketMicros的更新工作，这样减少了线程使用**， 节约了资源，并且也简化了操作。也就resync这个方法所做的事情。



```java
//根据现在的时间更新桶里令牌数和下一次获取令牌的毫秒
void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      storedPermits = min(maxPermits, storedPermits + newPermits);
        //更新nextFreeTiecket为now
      nextFreeTicketMicros = nowMicros;
    }
  }
```

resync()具体逻辑是：如果现在的毫秒（比如40ms）大于下次能授权的毫秒数（比如20ms），说明这个限流器已经有一段时间没有使用了，需要计算这段时间产生的令牌数。否则说明这段时间限流器一直有请求进来，则不需要更新.





### 稳定和渐进模式

Guava有两种限流模式，一种为稳定模式(SmoothBursty:令牌生成速度恒定)，一种为渐进模式(SmoothWarmingUp:令牌生成速度缓慢提升直到维持在一个稳定值)。还记得前面在看reserveEarliestAvailable时，这两种模式对storedPermitsToWaitTime有不同的实现吗？

```java
long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);
```



SmoothBursty的实现,等待毫秒数直接等于需要新增令牌数*生成一个令牌需要的时间

```java
@Override
    long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
      return 0L;
    }
```

SmoothWarmingUp的实现

```java
@Override
    long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
      double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
      long micros = 0;
      // measuring the integral on the right part of the function (the climbing line)
      if (availablePermitsAboveThreshold > 0.0) {
        double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
        // TODO(cpovirk): Figure out a good name for this variable.
        double length = permitsToTime(availablePermitsAboveThreshold)
                + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
        micros = (long) (permitsAboveThresholdToTake * length / 2.0);
        permitsToTake -= permitsAboveThresholdToTake;
      }
      // measuring the integral on the left part of the function (the horizontal line)
      micros += (stableIntervalMicros * permitsToTake);
      return micros;
    }
```



## 常见问题

### acquire和tryAcquire的区别

- acquire是阻塞的且会一直等待到获取令牌为止，它有一个返回值为double型，意思是从阻塞开始到获取到令牌的等待时间，单位为秒
- tryAcquire是另外一个方法，它可以指定超时时间，返回值为boolean型，即假设线程等待了指定时间后仍然没有获取到令牌，那么就会返回给客户端false，客户端根据自身情况是打回给前台错误还是定时重试

### RateLimiter的缺陷

特别注意RateLimiter是单机的，也就是说它无法跨JVM使用，设置的1000QPS，那也在单机中保证平均1000QPS的流量。

假设集群中部署了10台服务器，想要保证集群1000QPS的接口调用量，那么RateLimiter就不适用了，集群流控最常见的方法是使用强大的Redis：

- 一种是固定窗口的计数，例如当前是2019/8/26 20:05:00，就往这个"2019/8/26 20:05:00"这个key进行incr，当前是2019/8/26 20:05:01，就往"2019/8/26 20:05:01"这个key进行incr，incr后的结果只要大于我们设定的值，那么就打回去，小于就相当于获取到了执行权限
- 一种是结合lua脚本，实现分布式的令牌桶算法，网上实现还是比较多的，可以参考https://blog.csdn.net/sunlihuo/article/details/79700225这篇文章

总得来说，集群限流的实现也比较简单。

## 总结

推荐大家自己去读读这个google的手笔，从注释到命名一目了然，读起来很舒服。

其他收获：

静态工厂方法代替构造函数。`RateLimiter`是入口类，它提供了两套工厂方法来创建出两个子类。

最后以SmoothRateLimiter的一个可可爱爱注释收尾。

![1587137971779](https://wangxuanni.oss-cn-hongkong.aliyuncs.com/ratelimiter/%E6%B3%A8%E9%87%8A.png)