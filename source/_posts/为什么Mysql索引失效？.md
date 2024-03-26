

# 为什么Mysql索引失效？



上篇文章，我谈到了一次索引失效导致db打满的简单sql优化，准备写一篇关于索引失效的文章，在我写这篇文章的时候，[在MySQL索引失效原理是什么？](https://www.zhihu.com/question/421944348)问题下看的一个回答教人用口诀记住索引失效的回答：

> [索引失效](https://www.zhihu.com/search?q=索引失效&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2251613556})问题是面试必问问题，因为索引失效的情况比较多，很多同学记不住，面试的时候回答不好。我仔细研究了七七四十七天，设计了一句[七字口诀](https://www.zhihu.com/search?q=七字口诀&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2251613556})，记住这句口诀，以后再遇到这个问题就可以拿满分了。
>
> 七字口诀就是：
>
> **模 型 数 空 运 最 快**
>
> 口诀字面意思就是，要运送一个[产品模型](https://www.zhihu.com/search?q=产品模型&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2251613556})的话，要用空运，不要用陆运和海运，数空运最快。叫做：[模型数](https://www.zhihu.com/search?q=模型数&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2251613556})空运最快。
>
> 下面我拆开逐字讲解一下：
>
> **模**：模糊查询的意思。like的模糊查询以%开头，索引失效。

我看到这个答案不禁哑然失笑，这个口诀像老师明天要抽查今天死记硬背的学生用的招数，突击一下面试或许可行。但根据我的经验，更好的方式应该是知道是什么原理导致了这个规则，死记硬背背漏背错一点就差之千里。并且如果条件稍微变换，背的规则就不适用了。让我来逐条解释为什么索引会失效。





## 模糊查询中，通配符在最前面时，即LIKE '%abc'这样不能命中索引



索引失效的情况之一：

- **模糊查询中，通配符在最前面时，即LIKE '%abc'这样不能命中索引**

- **联合索引中，没有遵循最左匹配原则，联合索引会失效，又称之为最左匹配原则。**

- **联合索引中，遇到范围查询时，其后的索引不会被命中。**

  

这几条原理其实是类似的，LIKE '%abc'之所以失效，是因为在Mysql为字符串字段索引时，字符串的**字典序**排列的。什么是字典序呢？就是先比较第一个字符谁大，如果第一个字符同样大，再比较第二个字符，依次类推到最后一个字符。当你使用这样LIKE '%abc'或者这样的LIKE '%abc%'的字符，由于没有第一个字符，Mysql无法根据字典序比较索引去缩小范围，所以索引会失效。

而之所以联合索引有最左匹配原则，也是因为联合索引的排列规则。联合索引是先按左边第1列排列，如果左边第1列值相同再按左边第2列排列，以此类推。联合索引是在找到第一列的情况下才能找第二列，不能跳过左边的列找下一列。

所以用联合索引的第一列范围查询可以用到索引吗？第二列、第三列呢？比如建立联合索引id_a_b_c,下列sql那些条件可以走索引

```
SELECT * FROM testTable WHERE a > '10' AND b > '100'AND c < '20'

```

回答是：第一a列可以，后面的都不行。如果知道联合索引的建表原理就很容易知道为什么，因为只有在第一列相同的情况下才能使用第二列，**联合索引是根据第一列排列的，只有第一列能用范围查询**，右边的列都是寄存于左边的列排序的。

再延伸一个问题，如果第一列使用了精准匹配，第二列的范围查询可以使用到索引吗？

```
SELECT * FROM testTable WHERE a = '10' AND b > '20' AND  c > '100';

```

答案，是可以的，在a列相同的情况下，是根据b列排序的，但在这个sql里c列就不行了

可见这两个规则都是因为索引排列的原理，换句话说，索引按照一个顺序排列后面使用的时候，也必须按照这个这个排列规律去查询。

如果死记硬背最左匹配原则，那么条件稍稍变化就无从应对了。



## 对列进行函数运算的情况（如 where md5(password) = "xxxx"）

这个规则解释起来也简单，因为使用函数修饰过的列就不是原本建索引的那个列，值已经发生变化，无法使用索引比较查找了。



## 使用了not in, <>,!=则不会命中索引。注：<>是不等号

这是为什么呢，还是那两个字：**成本**。[上篇关于sql优化的文章](https://zhuanlan.zhihu.com/p/656972135/preview?comment=0&catalog=0)提过，**当Mysql发现通过索引扫描的行记录数超过全表的30%时，优化器就会放弃走索引，变成全表扫描。**一般来说，不等于某个值基本等于全表扫描，那还走什么索引。

这里有一个有意思的问题，如果不等于命中了大部分数据会不会走索引呢？我做了一个实验，表记录总共有150个，a字段只有11个记录是有值的，其他139个都是空字符串。先分析一个不等空的试试。

```
explain  SELECT * FROM `TestTable` WHERE `a`!=''
```

| id   | select_type | table     | partitions | type  | possible_keys | Key   |      |
| ---- | ----------- | --------- | ---------- | ----- | ------------- | ----- | ---- |
| 1    | SIMPLE      | TestTable | NULL       | range | idx_a         | idx_a |      |

| key_len | Ref  | Row  | Filtered | Extra                 |
| ------- | ---- | ---- | -------- | --------------------- |
| 258     | NULL | 12   | 100      | Using index condition |

可见使用不等于符号在索引可以缩小范围至低于30%的情况下是会走索引的。分析不使用不等于的sql，发现反而会索引失效。

```
explain  SELECT * FROM `ActivityInfo` WHERE `playbackLiveIds`=''
```

| id   | select_type | table     | partitions | type | possible_keys | Key  |      |
| ---- | ----------- | --------- | ---------- | ---- | ------------- | ---- | ---- |
| 1    | SIMPLE      | TestTable | NULL       | ALL  | idx_a         |      |      |

| key_len | Ref  | Row  | Filtered | Extra       |
| ------- | ---- | ---- | -------- | ----------- |
| Null    | Null | 150  | 92.67    | Using where |

**由这个实验，可见即便条件里有不等于符号，但不等于的条件命中了大部分数据，也会走索引。mysql是根据成本决定索引会不会失效的。**

