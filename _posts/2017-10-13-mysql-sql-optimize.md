---
title: MySQL几个简单SQL的优化
layout: post
guid: urn:uuid:8d2f2b63-g930-3d54-8ca7-fbvbds2a4fge
tags:
  - mysql
---

最近在做项目的时候，遇到了一些大数据量的操作，有大批量的CRUD的操作，一开始的实现的方案经过性能测试，发现性能并不是很好，然后开始审查代码，对相关可以提升性能的操作进行了优化，这里分享给大家。

### 原则

首先我这里不讲索引相关的内容以及数据库相应参数的优化，这里假设你对索引已经有了相关的了解了，我总结了下我这次的优化，主要两个原则：

- 一些特定的场景，尽量用批处理处理数据，比如批量添加数据，批量修改数据；
- 结合业务尽量减少SQL的执行次数和查询不必要的数据；

### 场景实践

为模拟运行场景，我这里建了一个表，并往里面添加了300w条数据，表结构如下：

```sql
CREATE TABLE `tb_big_data` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `weixin_id` varchar(64) NOT NULL,
 `openid` varchar(64) NOT NULL,
 `status` int(3) NOT NULL,
 `gmt_create` datetime NOT NULL,
 `gmt_modified` datetime NOT NULL,
 PRIMARY KEY (`id`),
 KEY `weixin_id_gmt_create_openid` (`weixin_id`,`gmt_create`,`openid`)
) ENGINE=InnoDB AUTO_INCREMENT DEFAULT CHARSET=utf8
```

#### 1.分页查询小优化

分页查询老生常谈，网上各种优化方法都很多，这里就不提及了，这里只是分享一个小技巧：

**如何在使用最普通的limit的时候提高性能？**

假设我们现在有一条这样的SQL：

```sql
SELECT * FROM `tb_big_data` where weixin_id ='gh_266a30a8a1f6' and gmt_create > '2017-10-10 00:00:00' order by id asc limit 800000, 100;

执行时间：100 rows in set (1.53 sec)
```

假如我们现在不能进行其他优化，比如传入最小id，分表查询等策略，以及不进行SQL预热，怎么提高这条SQL的速度呢？
其实很简单我们只需要一个in操作即可：


```sql
SELECT * FROM `tb_big_data` t1 where t1.id in ( 
    SELECT tt.id FROM ( 
        SELECT id FROM `tb_big_data` t2 where weixin_id = 'gh_266a30a8a1f6' and gmt_create > '2017-10-10 00:00:00' order by t2.id asc limit 800100, 100
        ) as tt);

执行时间：100 rows in set (1.17 sec)
```

可以看出只需稍加修改，SQL的效率可以提高30%~40%，而且在单条数据记录越大的情况下效果越好，当然这不是最好的分页方法，这只是一个小技巧；

#### 2.减少SQL查询

现在有一个需求我们现在有一个用户的列表（用户的唯一标识为openid）然后我们需要判断用户在当天是否有过相应的记录；

这是问题其实很简单，我们首先一想到的操作就是循环这个列表一个一个判断，很简单也很好实现，但是真正测试的时候发现性能却很差，尤其在数据量大的情况下，倍数级增长，这里有有网络数据传输消耗的时间和SQL本身的执行时间；

假设我们现在执行一条以下的SQL：

```sql
SELECT * FROM `tb_big_data` WHERE weixin_id ='gh_266a30a8a1f6' and gmt_create > '2017-10-13 00:00:00' and openid='2n6bvynihm5bzgyx';

执行时间：1 row in set (0.95 sec)

```

现在如果我们执行100次，不敢想象会是什么情况，庆幸自己发现了这个问题，因为在数据量少的情况下，这个问题表现的并不是那么严重，其实我们稍加改变就能以另一种高效的方式解决这个问题：

```sql
SELECT * FROM `tb_big_data` WHERE weixin_id ='gh_266a30a8a1f6' and gmt_create > '2017-10-13 00:00:00' and openid in ('2n6bvynihm5bzgyx','1stbvdnl63de2q37','3z8552gxzfi3wy27'...);

执行时间：100 row in set (1.05 sec)
```

发现了没有，还是用in，而且执行时间几乎与单条查询的时间一样，可见只是单一这一部分处理就可以提升了很大的性能。

#### 3.特定场景使用SQL的批处理

这个跟上一点有一个相似点，那就是减少SQL执行，上面只是查询而已，而当出现大批量的CUD的操作时，执行每条SQL，数据库都会进行事务处理，这将会消耗大量的时间，而且极端情况下会引起大批量SQL等待无法执行，导致业务出错，正是因为这些原因，我们在一些适当的情况下可以使用批处理来解决这个问题。

##### （1）批量插入

批量插入比较简单，也比较常用，这里就给一下基本语法：

```sql
INSERT INTO table_name (field1,filed2,...) values (value11,value12,...),(value21,value22,...),...
```

##### （2）批量更新

我先举个简单的例子，我们现在来根据一些条件来更新数据，具体SQL如下：

```sql
update `tb_big_data` set status = 2 WHERE weixin_id ='gh_266a30a8a1f6' and gmt_create > '2017-10-13 00:00:00' and openid = '2n6bvynihm5bzgyx';

Query OK, 1 row affected (2.28 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
很惊讶，我们只是更新了一条记录，而且更新条件上是有复合索引的，没想到速度还那么慢，可以想象如果我们批量更新数据，那得耗时多少；

但是我们看一下另一条SQL：

```sql
update `tb_big_data` set status = 1 WHERE id = 900098;

Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

上面的id值为之前条件筛选出来的记录的id，是不是很惊讶，怎么这条SQL执行的时间几乎不需要什么时间，所以我们可以利用这个特点和批量查询简化批量更新，虽然这种方式不能让性能到最优，但是也能提升很大了，我进行了一个测试，根据相应条件批量更新100条数据：

方式|直接批量更新|先批量查主键再批量更新
---|---|---
耗时 | 289.12s | 1.342s

可以看出这种方式相对对于普通方式来说，性能提升巨大，具体执行的时候我们也可以将这些SQL放在一个事务提交，减少数据库事务次数，但只这是一种在代码层面上的优化；

另外我们可以利用MySQL提供的特殊语法进行批量更新，具体语法为：

```sql
#语法
INSERT INTO table_name (id,field1,field2,...) VALUES  (id1,value11,value12,...),(id1,value11,value12,...),... on duplicate key update  field = VAULES(field);

#使用例子

INSERT INTO `tb_big_data` (id,weixin_id,openid,gmt_create,status) values  (1,'gh_266a30a8a1f6','w9q8fmodytjgppsr','2017-10-13 12:00:00',3),(2,'gh_266a30a8a1f6','bu1flmch4i8eegzf','2017-10-13 12:00:00',3) on duplicate key update status = VAULES(status);

```

经过测试这种方式在数据量小的情况下与上述方式效率差不多，但是随着数据量越来越大，性能也越来越好，缺点的话主要传输的数据量很大，不需要更新的字段也需要传输。

另外也不推荐大量数据的批量更新，一次不要超过1000条为好。

### 总结

总的来说，SQL优化是一门细心的学问，需要不断去尝试，测试，找到最优方式，另外还有一点就是要结合实际情况，综合考虑选择合适的方式。