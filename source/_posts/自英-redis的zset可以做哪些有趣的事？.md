---
title: What interesting things can zset for redis do?
date: 2022-07-26 00:24:50
categories: redis
description: Multidimensional leaderboard and rate Limiting.

---





## Foreword



I have always had the habit of doing work summaries, and I will occasionally carry out an extended study of the technologies I use in my work, but the summary notes are written for myself. some. I happened to be learning English recently, so I thought I might as well organize my notes in English. This is the first work note I have compiled in English。

 I recently developed a timed spike function using redis' zset. The specific implementation is to use the seckill time as the score of the zset, use the product ID as the value, the key is the name of the seckill product set, and use a timed task to scan the zset at the second level, start is a very early time, and end is the current timestamp , use the method zrangeByScore to get the commodities of this period. If the seckill time of a commodity falls within this range, it will execute the opening seconds. After this function is completed, I want to summarize zset, which can achieve many interesting functions, so I have this article. 



## What interesting things can zset do? 



### Multidimensional leaderboard 

The most common usage is to implement a leaderboard, such as a list of winning countries based on the number of Olympic medals. Here I would like to introduce a more interesting implementation - multi-dimensional leaderboard. The multi-dimensional ranking means that the rankings are ranked according to multiple different dimensions. For example, for example, in the medal rankings of Olympic countries, the participants are ranked from the dimensions of gold, silver and bronze medals, and the number of gold medals is first sorted. ; If the number of gold medals is the same, then sort by the number of silver medals.

Isn't it easy to think of piecing together two numbers as a fraction? But this simple way of putting together is problematic. For example, if country A has won 5 gold medals and 15 silver medals, the combination is 515; another country B has won 50 gold medals and only 1 silver medal. , which when pieced together is 501, yields a higher score for the A country, when the opposite is true. 

There are two ways to implement multidimensional indexing：

#### Binary segmentation

The reason for the failure of direct piecing is essentially because the low-order bits grabbed the high-order bits, so we can distinguish the low-order bits from the high-order bits, and fill in zeros in front of the high-order bits that are not filled. Fractions of different dimensions can be stored in different binary digits of a variable. This kind of binary segmented storage design idea is very common in Java source code. For example, in the design of thread state in Java's thread pool, the thread state is stored in AtomicInteger. This AtomicInteger traverses the lower 29 bits to indicate the number of threads in the thread pool. , through the upper 3 bits to indicate the running status of the thread pool. 

#### Split into decimals 

Or we can directly split into decimals in multiple dimensions. For example, the gold medal is an integer, and the silver medal is placed in a decimal place. If country A has 5 gold medals and 15 silver medals, the combination is 5.15; another country B has 50 gold medals and only 1 silver medal, and the combination is 50.1. 

### Rate Limiting

#### counter 

If you want to use redis to implement a current limit, first of all think that you can implement counter current limit based on Redis's setnx. For each request, use the unique identifier of the requested resource as the key. If the key already exists, it will be incremented by 1. If the initialization counter is not created, it will be 1. According to the size of the counter, it is judged whether the current limit threshold is exceeded, and if it exceeds, it will return to trigger the current limit. 

Simple counter current limiting has a fatal flaw, the criticality problem. For example, in a scenario where the total number of requests is limited to 100 in 1 minute, 100 requests came in the first minute until the end of the minute, and 100 requests came immediately at the beginning of the next minute. Because it is in two different one-minute intervals, the counter current limit will be released, but in fact, 200 requests came in less than one minute, violating the current limit rules. Therefore, the counter current limit is generally not used by anyone. 

#### sliding window 

Sliding window current limiting can solve this critical problem. To implement sliding window, we have to use zset. The principle is similar to that of a timed spike. The timestamp is used as a score, and zset can be used to obtain the corresponding value according to the range of the score. The specific implementation is as follows：

When the request comes, the unique identifier of the requested resource is used as the key. When each request comes in, the score of the zset is the current timestamp, and the value of the zset remains unique and can be generated by UUID; 

The beginning of the sliding window: the current time - the time period of the current limit, the beginning of the sliding window: the current time. 

Use zset's zrangebyscore to get how many requests are in the sliding window. If the size of the threshold has been exceeded, the current limit is triggered. Otherwise, add a new piece of data in zset. 

In order to avoid the infinite increase of zset, regularly use zremrangebyscore to delete data smaller than the end of the sliding window, because the data beyond the window does not care. 

Note: The difference between zrange and zrangebyscore, zrange divides the interval according to the index, and zrangebyscore is according to the score.

A simple code example is as follows：

```java
public class RedisSlidingWindowRateLimit {
  
    private static final int LIMIT = 100;
  
    private static final int PERIOD = 60;
    @Resource
    private RedisService redisService;

    public boolean triggerLimit(String requestPath) {
        long now = System.currentTimeMillis();
        Set<String> range = redisService.opsForZSet().rangeByScore(requestPath, now - PERIOD * 1000, now);
        if (range.size() > LIMIT) {
            return true;
        } else {
            redisService.opsForZSet().add(requestPath, UUID.randomUUID().toString(), now);
            return false;
        }
    }
}
```




