---
title: java内存泄露事故分析
date: 2022-10-10 14:06:33
tags: [bug]
categories: [开发日常]
---

### 问题描述

**电影票**长期存在**ticket**模块堆栈内存溢出导致系统崩溃的问题，初期一直认为是订单量过大导致比价功能调用次数太多引起的内存溢出问题。经过排查dump日志未能定位到具体问题，最终特权选择多开**ticket**模块的方式来避免单机崩溃导致服务不可用的情况，但是此方法治标不治本，运维需要经常重启崩溃的**ticket**模块。

> *电影票ticket模块在h5用户下单时，会调用异步比价功能，此功能使用@Async注解实现，意味着每有一笔订单都会启用一个比价线程*

本问题历时一年才得以解决，这其中经历了四个问题分析阶段

1. **ticket**模块崩溃，根据dump日志发现内存中存在大量比价功能生成的对象，初步判断是短时订单量过大导致的，未有较好的解决办法，选择多开**ticket**模块顶住大量订单的压力。
2. 运维侧报告**ticket**模块存在凌晨崩溃的情况，排查日志发现凌晨订单只有几笔，此时怀疑是比价功能存在一定问题，导致其产生的对象停留较久，但是依旧未排查到问题。

	> *因还是未定位到能导致内存溢出的问题，只能判断认为是虚拟机内存回收较慢导致的*

3. 技术侧发现`@Async`注解支持选择线程池来将异步线程改为用线程池来执行，可以防止大量调用`@Async`导致生成了海量线程从而内存溢出。但是在本功能上线后，依然出现了**ticket**模块崩溃的问题。
4. 经过以上三次排查，已基本确定本次问题是由代码问题引起的内存泄露，并不是因为电影票订单数据量过大导致的系统崩溃。随后重新开始排查问题，并最终定位到是因为json序列化引起的内存泄露。

### 定位思路

#### 步骤1
首先使用**JProfiler**工具查看**dump**日志，按照保留大小倒序排序，可以发现最多的是`String`对象，这是因为`YtbShowListResponse`内存溢出对象中存储有请求接口返回的字符串内容，此内容包含一个影院下面的所有场次，所以内容较大。

![avatar](JProfiler1.png)
<center>
（JProfiler工具查看dump日志）
</center>

<!-- more -->

#### 步骤2
查看对象的数据结构，发现本质是一个`Map`，初步怀疑是不是**NutMap**第三方工具引起的

![avatar](YtbShowListResponse.png)
<center>
（YtbShowListResponse数据结构）
</center>

#### 步骤3
使用**JProfiler**工具的图表功能，查看对象到GC根(root)的路径，可以发现最终是被**org.nutz.lang.Mirror**对象所持有，导致无法被回收

![avatar](JProfiler2.png)
<center>
（JProfiler工具查看图标）
</center>

#### 步骤4
此时查看比价功能代码，可以发现`YtbShowListResponse`对象的使用都是正常的，并没有放置到java静态变量中等操作，但是根据**dump**日志分析其中的`YtbShowListResponse`对象，`YtbShowListResponse`中`content`字段都是成功，即请求失败就不会出现内存泄露问题，问题代码范围进一步缩小。

![avatar](queryYtbOrderInfo.png)
<center>
（比价功能代码）
</center>

#### 步骤5
排查代码发现***getResponse***方法有非常高的可疑性，使用了第三方序列化方式***Json.fromJson***，并且使用了**PType**第三方工具来标识序列化类型，更加可疑，此序列化方法和**org.nutz.lang.Mirror**同属于**nutz**第三方工具包，基本已锁定到问题代码。

``` java
	@Override
    public List<YtbShowBo> getResponse() {
        String data = this.getData();
        if (Strings.isBlank(data)) {
            throw new YtbException("影托邦场次列表响应为空");
        }
        return Json.fromJson(new PType<List<YtbShowBo>>() {
        }, data);
    }
```

#### 步骤6

为了验证是否是json序列化导致，在`YtbShowListResponse`上重写了***finalize***方法，并在其中打印了日志，验证对象是否被回收了，采用对照法，使用**fastjson**包替换了序列化方法，执行相同比价功能，最终验证结果如下：

1. ***Json.fromJson***方法和**PType**共同使用情况下，会导致***finalize***不被执行，也就意味着此对象不会被虚拟机回收
2. ***JSON.parseObject***方法和**TypeReference**共同使用情况下，***finalize***很快就执行了，意味着对象被回收了

> java的***finalize***方法在生产环境是禁止使用的，**对其的重写可能会造成虚拟机性能下降或者对象无法被回收的问题**，最终导致系统崩溃（网上说的，未验证ㄟ( ▔, ▔ )ㄏ），本次只是在测试环境打印日志，方便检查对象是否被回收，**切勿在生产环境使用本方法**。

> *其实java虚拟机垃圾回收执行的很快*

#### 总结

到这里，其实问题已经得到了解决，但是为了提现我们热爱技术的一面，也为了彰显我们的钻研精神，那么还要提出疑问，为什么这会导致内存泄露？总结出以下几个问题：

- 为什么**fastjson**不会出现内存泄露的问题？
- 明明序列化的对象是字符串**data**数据，生成后的对象也是`List<YtbShowBo>`，怎么结果就成了`YtbShowListResponse`存到了虚拟机中无法被回收？
- json序列化的原理是什么？

本着以上几个问题，我们进入了下一阶段，我们要找到问题本质~我们要搞清楚到底是什么东西，影响了虚拟机内存回收？

### 追本溯源

进入**nutz**的jar包源码，可以发现他使用了`mirrorCache`把所有使用到的序列化类型都缓存了起来，问题就出在了这里

![avatar](Mirror.png)
<center>
（json序列化源码）
</center>

这时就要开始使用我们的好帮手百度了，**PType**实现了**ParameterizedType**接口，使用**ParameterizedType** + 内存溢出为标题查到一篇文章<sup>①</sup>，说明**fastjson**也存在使用**ParameterizedType**不当导致的内存泄露问题，此问题已经被**fastjson**修复了。不过**nutz**包和**fastjson**包并不相同，但是这也印证了我们的观点，json序列化如果使用不当确实会导致内存泄露的问题。

> **ParameterizedType**是jdk提供的用于处理带泛型类型的类的序列化方式<sup>②</sup>

比对了**nutz**包和**fastjson**的源码，发现了区别，**fastjson**并没有直接使用序列化类型**Type**，而是在其内部进行了重新实现，到这里，我们就能回答出前面三个问题中的两个了：
- **fastjson**采用了在**Type**中重新生成子序列化类型的方式，避免了内存泄露的问题
- json序列化的原理就是通过声明一个序列化类型，然后将字符串内容进行转换的过程

![avatar](nuztAndFastjson.png)
<center>
（**nutz**包和**fastjson**源码比较，左**nutz**，右**fastjson**）
</center>

但是，第二个问题没有得到解答，为什么最终在内存中存储了大量的`YtbShowListResponse`而不是`List<YtbShowBo>`呢？

这个问题最终通过研究`new TypeReference`和`new PType`两者之间的区别得以解决

先看下声明序列化类型的写法, `new xxx(){}`是java匿名内部类的写法。

``` java
        List<YtbShowBo> ytbShowBos = Json.fromJson(new PType<List<YtbShowBo>>() {
        }, data);
```

此时，你只需要回想下学过的java基础，就能明白一点，java的匿名内部类是依赖于其父类的，这就导致你在dump日志或者debug模式下查看匿名内部类对象时，都会惊奇的发现他的指针并不是指向他的内存地址，而是直接指向其父类

> *匿名内部类其实极易导致内存泄露问题<sup>③</sup>*

![avatar](debug.png)
<center>
（debug）
</center>

到这里，你也就明白了为什么会出现内存泄露的问题，为什么内存中会有大量的`YtbShowListResponse`对象，因为每一个`new PType`都持有了其父类对象，当**PType**被放置到`mirrorCache`中缓存起来时，也就导致他持有的父对象再也无法被回收了，我们也就解答了前面第二个问题，也就明白了这次内存泄露的本质

##### 这就结束了吗？

问题找到了，但是我们还没有去解决问题，前面好像已经找到了解决办法，就是替换第三方序列化工具，使用**fastjson**替代**nutz**，但是这样好像我们学到的知识没用上，如何使用java基础解决这个问题？让我们倒回前文有关匿名内部类的部分，解决办法就藏在这里面 ↑

##### 终极解决方案：

将在方法内部创建的**Type**改为静态变量

`private static final PType<List<YtbShowBo>> YTB_SHOW_BO_TYPE = new PType<List<YtbShowBo>>() {};`

本问题随即解决

> 当匿名内部类成为静态变量时，根据我们学过的java基础知识，java类下的静态变量属于类，被所有类实例对象所共享，当且仅当在类初次加载时会被初始化

##### 到此为止了，但是没有完全为止

本次内存泄露问题已经圆满解决，大家也跟着本文档学习了解了如何排查问题，如何分析问题，如何解决问题。但是本文档只是个引子，希望在以后的工作中大家都能保持钻研精神，不怕问题，解决问题。

### 附录

收录相关在排查问题时查询到的有参考意义的博客内容

附录① [Java反序列化json内存溢出_fastjson反序列化使用不当导致内存泄露](https://blog.csdn.net/weixin_33358099/article/details/114828657)

附录② [ParameterizedType详解](https://blog.csdn.net/JustBeauty/article/details/81116144)

附录③ [JVM--Java内存泄露--匿名内部类--原因/解决方案](https://blog.csdn.net/feiying0canglang/article/details/124946774)