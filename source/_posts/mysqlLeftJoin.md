---
title: mysql中多表LEFT JOIN的误区
date: 2022-11-12 17:06:47
tags: [mysql]
categories: [mysql]
---

最近在日常看项目慢SQL日志的时候，发现项目里的用户管理查询SQL略慢，基本要在一秒以上，决定优化一下，结果优化了半天还优化出来个问题，研究了好久才发现原因，其实是我的使用姿势不对

#### 背景介绍

用户列表查询使用了多表关联，其中使用了7张表做LEFT JOIN关联查询，拼接了很多查询条件，并且还穿插了几个子查询。所以做了一下优化，重新梳理了表逻辑

其中有三张表分别是`user`用户表，`user_brand`用户品牌表，`user_add_account`用户加款账户表，查询逻辑是

``` SQL
SELECT u.name, group_concat(ub.name) brandName, group_concat(uac.account) account
FROM `user` u 
LEFT JOIN `user_brand` ub ON u.id = ub.user_id 
LEFT JOIN `user_add_account` uac ON u.id = uac.user_id
```

结果出现了brandName和account相同内容重复的问题，之前brandName是通过子查询得出的，所以没有存在重复的情况，结果改成LEFT JOIN出现了重复问题，这个问题也是研究了好久才发现，其实是因为对SQL查询原理掌握不牢导致的

<!-- more -->

#### 原因

三表的关系是，`user`和`user_brand`是一对多的关系，`user`和`user_add_account`也是一对多的关系，在mysql中，如果多表使用LEFT JOIN来进行关联查询，并且存在多个左表和右表的一对多关系，mysql会将多个右表的条数进行笛卡尔积对应到左表上，就导致了`user_brand`和`user_add_account`都存在重复出现的情况

##### 问题SQL

``` SQL
SELECT u.id, ub.brand_id, uaa.account 
FROM `user` u 
LEFT JOIN `user_brand` ub ON u.id = ub.user_id 
LEFT JOIN `user_add_account` uaa ON uaa.user_id = u.id 
```

##### 查询结果

| id | brand_id | account |
|---- | ---- | ---- |
| 8 | 1 | 5719xxxxxxxx901 |
| 8 | 1	| 5719xxxxxxxx555 |
| 11 | 1 | 1209xxxxxxxx902 |
| 11 | 2 | 1209xxxxxxxx902 |
| 11 | 3 | 1209xxxxxxxx902 |

很明显可以看出，id为8的数据因为`user_add_account`存在两条数据，导致会查出来两条数据，id为11的数据因为`user_brand`存在三条数据，导致会查出来三条数据，假如有一个id为20的数据，关联到`user_brand`2条，关联到`user_add_account`三条，那最终会查出6条数据

#### 怎么解决？

两种方案，第一种可以在group_concat中加DISTINCT关键词去重，因为实际业务查询只是为了看关联到user_brand的数据，去重并不会影响业务的展示

第二种可以去除LEFT JOIN关联的表，采用单独查询后拼接的方式，也能避免本情况，顺便还能减少表关联，减轻数据库查询压力