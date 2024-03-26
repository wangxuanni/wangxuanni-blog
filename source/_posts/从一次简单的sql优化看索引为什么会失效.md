# 从一次简单的sql优化看索引为什么会失效

## 问题

某日DBA找到我，说某条sql打满了数据库的磁盘。sql如下：

```sql
select * from MaterialInfo
WHERE
  (
    mIds <> ''
    and itemIds = ''
    and deleted = 0
  )
order by
  id desc
limit
  1000

```

该表是有联合索引`idx_itemIds_mIds`，按sql的条件是会走索引的。那到底是什么原因导致全表扫描了。难道是因为mIds使用了不等于这种条件？不对，因为联合索引会先根据最左边的itemIds去匹配。那是因为是itemIds查询空字符，所以没有走索引？这里先留一个悬念。且看我娓娓道来。

## 分析

话不多说，我们来分析一下，在sql前加上explain关键字就可以分析该sql的执行计划啦

```sql
explain select * from MaterialInfo
WHERE
  (
    mIds <> ''
    and itemIds = ''
    and deleted = 0
  )
order by
  id desc
limit
  1000

```



下面是执行计划：

| id   | select_type | table        | partitions | type  | possible_keys    | Key      |
| ---- | ----------- | ------------ | ---------- | ----- | ---------------- | -------- |
| 1    | SIMPLE      | MaterialInfo | NULL       | Index | idx_itemIds_mIds | PROMARRY |



| key_len | Ref  | Filterd | Filtered | Extra       |
| ------- | ---- | ------- | -------- | ----------- |
| 8       | NULL | 2000    | 4.5      | Using where |



啥？看不懂这一坨是什么？不慌，在简单的单表查询，一般我们只需要关注其中的四个重要字段就可以了。possible_keys是可能用到的索引，key是实际上使用的索引，rows是预估的需要读取的记录条数，filtered是某个表经过搜索条件过滤后剩余记录条数的百分比。

为什么可能使用到的索引（possible_keys）是联合索引idx_itemIds_mIds，而实际上用的到索引（key）是主键索引呢？possible_keys只是Mysql列出可能用到的索引，实际上用哪个会根据成本具体判断。我查了一下itemIds为空数据占总数据的**3/4**。**当Mysql发现通过索引扫描的行记录数超过全表的30%时，优化器就会放弃走索引，变成全表扫描。**在这个sql里，Mysql分析使用联合索引，发现itemId为空就能查出3/4，就直接放弃了走联合索引idx_itemIds_mIds。

## 行动

我顺着这条sql去找，找到了罪魁祸首——一个定时任务。请看脱敏的代码

```java
MaterialDOExample query = new TrafficMaterialDOExample();

query.createCriteria().andMIdsNotEqualTo("").andItemIdsEqualTo("").andDeletedEqualTo(0);

query.setOrderByClause("id desc");

//查询ItemIds为空，mId为空的记录

List<MaterialDO> materialDOS = materialInfoMapper.selectByExample(query);

materialDOS.forEach(trafficMaterialDO -> {

  //省略了一部分代码，意图是把记录里的extra解析处理，设置回记录里去

  MaterialDO materialDO = new TrafficMaterialDO();

  materialDO.setId(trafficMaterialDO.getId());

  materialDO.setItemIds(itemIds);

  materialInfoMapper.updateByPrimaryKeySelective(materialDO);

  *logger*.info("Task success:{}", materialDO.getId());

});
```



这个任务简而言之是找到符合条件的记录做加工。具体来说是去找到ItemIds为空，mId不为空的记录做解析，解析完成后再更新到db里去。是有意去这么设置条件的。

**如果意图是定时去扫描并处理表中某字段为空的，应该怎么写比较好呢，其实思路就是，只查增量数据，用其他字段走索引。**

想了两个方法，我后面采用了方法二去优化这个定时任务

方法一 如果表有插入时间这个字段并且有索引，那么可以走插入时间字段的索引，比如30min执行一次的任务，就查找半小时前~当前时间的所有新增数据有没有需要处理的。

方法二 如果表没有插入时间这个字段，那么可以走id这个索引。查出当前最大的id，去缓存取出最近一次执行最后id，查出所有>最近一次id并且<当前最大的id的记录,进行处理，完事后将当前最大id更新进缓存。

经过这次优化，我对索引什么时候会失效的情况产生了兴趣。下一篇文章，我会用原理将清楚各种索引失效的原因。

## 总结：为什么索引会失效？

在我写这篇文章的时候，看到了知乎一个回答教人用口诀记住索引失效的情况：



1. 模糊查询中，通配符在最前面时，即LIKE '%abc'这样不能命中索引

2. 使用了not in, <>,!=则不会命中索引。注：<>是不等号

3. innoDB引擎下，若使用OR，只有前后两个列都有索引才能命中（执行查询计划，type是index_merge），否则不会使用索引。

4. 对列进行函数运算的情况（如 where md5(password) = "xxxx"）

5. 联合索引中，遇到范围查询时，其后的索引不会被命中。或者没有遵循最左匹配原则。

   

我看到这个答案不禁失笑，这个口诀真像老师明天要抽查今天死记硬背的学生用的招数。更好的方式是知道是什么原理导致了这个规则，死记硬背容易忘。并且如果条件稍微变换，背的规则就不适用了。最后让我来逐条解释为什么索引会失效。

模糊查询中，通配符在最前面时，即LIKE '%abc'这样不能命中索引

联合索引中，遇到范围查询时，其后的索引不会被命中。或者没有遵循最左匹配原则。

联合索引中，遇到范围查询时，其后的索引不会被命中。或者没有遵循最左匹配原则。

这两条原理其实是类似的，LIKE '%abc'之所以失效，是因为在Mysql为字符串字段索引时，字符串的**字典序**排列的。什么是字典序呢？就是先比较第一个字符谁大，如果第一个字符同样大，再比较第二个字符，依次类推到最后一个字符。当你使用这样LIKE '%abc'或者这样的LIKE '%abc%'的字符，由于没有第一个字符，Mysql无法根据字典序比较索引去缩小范围，所以索引会失效。

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



使用了not in, <>,!=则不会命中索引。注：<>是不等号

这是为什么呢，还是那两个字**成本**，**当Mysql发现通过索引扫描的行记录数超过全表的30%时，优化器就会放弃走索引，变成全表扫描。**一般来说，不等于某个值基本等于全表扫描，那还走什么索引。

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





对列进行函数运算的情况（如 where md5(password) = "xxxx"）



这个规则解释起来也简单，因为使用函数修饰过的列就不是原本建索引的那个列，值已经发生变化，无法使用索引比较查找了。