---
title: java面试题汇总
date: 2022-11-02 20:41:25
tags: [java]
categories: [面试题]
---

本文记录一些比较贴合实际开发的常遇到的java面试题，方便后续面试

### 【易】StringBuilder和StringBuffer区别是什么？

StringBuffer是线程安全的，StringBuilder是非线程安全的，但是StringBuilder性能高于StringBuffer，单线程下推荐使用StringBuilder，**StringBuffer内部采用synchronized来实现并发下的线程安全，也导致性能较于StringBuilder更低** *【加分项】*

### 【中】在 Java 程序中怎么保证多线程的运行安全？

- 使用安全类，比如 Java. util. concurrent 下的类
- 使用自动锁 synchronized
- 使用手动锁 Lock
- volatile关键字

### 【难】volatile关键字作用，如何实现的原子性、可见性、有序性

volatile只能保证可见性和有序性，对于一些复杂操作无法保证原子性

1. 可见性：

> CPU的运行速度是远远高于内存的读写速度的，为了不让cpu为了等待读写内存数据，现代cpu和内存之间都存在一个高速缓存cache，线程在运行的过程中会把主内存的数据拷贝一份到线程内部cache中，也就是working memory。这个时候多个线程访问同一个变量，其实就是访问自己的内部cache。这就导致当线程A把变量flag加载到自己的内部缓存cache中，线程B修改变量flag后，即使重新写入主内存，但是线程A不会重新从主内存加载变量flag，看到的还是自己cache中的变量flag。所以线程A是读取不到线程B更新后的值。

volatile会对修饰的变量添加LOCK指令，这个指令有两个最用

- 将当前处理器缓存的数据刷新到主内存
- 刷新到主内存时会使得其他处理器缓存的该内存地址的数据无效

这也就保证了变量的可见性

2. 有序性

计算机在执行程序的过程中，编译器和处理器通常会对指令进行重排序，这样做的目的是为了提高性能。而volatile通过借助内存屏障这个概念保证了操作的有序性。

volatile重排序规则：

- 当第一个操作是volatile读时，无论第二个操作是什么都不能进行重排序。
- 当第二个操作是volatile写时，无论第一个操作是什么都不能进行重排序。
- 当第一个操作是volatile写，第二个操作为volatile读时，不能进行重排序。

根据volatile重排序规则，Java内存模型采取的是保守的屏障插入策略，volatile写是在前面和后面分别插入内存屏障，volatile读是在后面插入两个内存屏障，具体如下：

- volatile读：在每个volatile读后面分别插入LoadLoad屏障及LoadStore屏障（根据volatile重排序规则第一条）
  - LoadLoad屏障的作用：禁止上面的所有普通读操作和上面的volatile读操作进行重排序
  - LoadStore屏障的作用：禁止下面的普通写和上面的volatile读进行重排序。

- volatile写：在每个volatile写前面插入一个StoreStore屏障（为满足volatile重排序规则第二条），在每个volatile写后面插入一个StoreLoad屏障（为满足voaltile重排序规则第三条
  - StoreStore屏障的作用：禁止上面的普通写和下面的volatile写重排序
  - StoreLoad屏障的作用：防止上面的volatile写与下面可能出现的volatile读/写重排序。

> 因为Java内存模型所采用的屏障插入策略比较保守，所以在实际的执行过程中，只要不改变volatile读/写的内存语义，编译器通常会省略一些不必要的内存屏障。

3. 原子性

为什么一些复杂操作无法保证原子性？例如i++，此操作实际可以分解为

- 线程读取i的值
- i进行自增计算
- 刷新回i的值 

会出现ab两个线程取到同一个i值，例如5，然后进行自增运算得到要刷新的值为6，这是a线程率先完成赋值操作，i变为6，随后b线程进行赋值操作，因b线程已完成自增运算，所以也会用得到的值6来进行赋值操作，从而导致不满足原子性

### 【难】synchronized的底层实现原理是什么？锁升级原理是什么？



<!-- more -->

### 【易】遍历List操作的几种方式？

- for循环遍历
- 增强for循环
- 迭代器循环
- jdk8后的lambda表达式，foreach方法 *【加分项】*

### 【中】遍历循环删除元素的话你会用那种方法来操作？

使用迭代器循环删除

### 【中】说一下 session 的工作原理？

session 的工作原理是客户端登录完成之后，服务器会创建对应的 session，session 创建完之后，会把 session 的 id 发送给客户端，客户端再存储到浏览器中。这样客户端每次访问服务器时，都会带着 sessionid，服务器拿到 sessionid 之后，在内存找到与之对应的 session 这样就可以正常工作了

### 【中】try-catch-finally 中，如果 catch 中 return 了，finally 还会执行吗？

finally 一定会执行，即使是 catch 中 return 了，catch 中的 return 会等 finally 中的代码执行完之后，才会执行

### 【低】spring 事务实现方式有哪些？

**声明式事务**：声明式事务也有两种实现方式，基于 xml 配置文件的方式和注解方式（在类上添加 @Transaction 注解）

**编码方式**：提供编码的形式管理和维护事务

### 【低】MySQL 的内连接、左连接、右连接有什么区别？

内连接关键字：inner join；左连接：left join；右连接：right join。

内连接是把匹配的关联数据显示出来；左连接是左边的表全部显示出来，右边的表显示出符合条件的数据；右连接正好相反。

### 【低】mysql默认事务隔离级别？

REPEATABLE-READ 可重复读

### 【中】乐观锁和悲观锁的区别？

**乐观锁**：每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在提交更新的时候会判断一下在此期间别人有没有去更新这个数据

**悲观锁**：每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻止，直到这个锁被释放

### 【难】如何保证接口幂等性？

{% note info %} 没有唯一正确的答案，更多地考察面试者的实际经验和总结能力 {% endnote %}

1. 定义唯一msgId，根据msgId判断是否重复
2. 结合业务逻辑使用乐观锁来限制，例如订单回调的业务逻辑，可以根据回调时订单的当前状态来判断本次回调内容是否要消费，并且在消费时借助乐观锁来进行限制，添加前置状态校验，类似
*【加分项】*
``` SQL
update `order` set `status` = 'status' where `status` = 'beforeStatus';
```

3. 每个请求操作必须有唯一的 ID，而这个 ID 就是用来表示此业务是否被执行过的关键凭证，例如，订单支付业务的请求，就要使用订单的 ID 作为幂等性验证的 Key；每次执行业务之前必须要先判断此业务是否已经被处理过；第一次业务处理完成之后，要把此业务处理的状态进行保存，比如存储到 Redis 中或者是数据库中，这样才能防止业务被重复处理

### 【难】JVM中有哪些垃圾回收算法？

### 【中】什么是Redis缓存穿透、击穿、雪崩？

**缓存穿透**：指访问一个缓存和数据库中都不存在的key，由于这个key在缓存中不存在，则会到数据库中查询，数据库中也不存在该key，无法将数据添加到缓存中，所以每次都会访问数据库导致数据库压力增大

> 1. 将空key添加到缓存中
> 2. 使用布隆过滤器过滤空key

**缓存击穿**：指大量请求访问缓存中的一个key时，该key过期了，导致这些请求都去直接访问数据库，短时间大量的请求可能会将数据库击垮

> 1. 添加互斥锁或分布式锁，让一个线程去访问数据库，将数据添加到缓存中后，其他线程直接从缓存中获取
> 2. 热点数据key不过期，定时更新缓存

**缓存雪崩**：指在系统运行过程中，缓存服务宕机或大量的key值同时过期，导致所有请求都直接访问数据库导致数据库压力增大

> 1. 将key的过期时间打散，避免大量key同时过期
> 2. 使用互斥锁重建缓存

### 【中】Redis怎么实现分布式锁？

使用命令setnx来实现，假如当前key有值则说明锁已经被占有，程序在使用完锁后需要调用del释放锁；

> redis的分布式锁会存在锁超时的情况，如果程序执行时间过长会导致锁自动解开，从而分布式锁就失效了，可以借助Redission的watchdog机制来解决；因为设置锁和设置过期时间是分开操作的，可能存在设置完锁服务宕机的情况，导致锁没有正确释放，这些可以通过lua脚本来实现，合并到一条命令中，也可以通过Redission来实现。*【加分项】*

### 【中】RPC请求，服务端在收到方法名、接口名、参数名后怎么完成一次调用？

服务端通过rpc解析到参数后，通过反射的方式，调用具体方法

### 【低】最近有在学习新的东西吗？有在了解哪块技术吗？

答案不限，关注面试者是否有自律性和学习能力