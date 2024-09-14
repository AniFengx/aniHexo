---
title: ClickHouse笔记
date: 2024-08-30 09:38:17
tags: ClickHouse
categories: [ClickHouse]
---

项目最近需要用到ClickHouse来做大数据的实时分析，但是ClickHouse相关文档过少，只能通过做笔记的形式手动记录

### 支持的数据类型

[参考地址](https://clickhouse.com/docs/zh/sql-reference/data-types "ClickHouse数据类型")

#### 数值类型:

+ Int8、Int16、Int32、Int64、Int128、Int256：有符号整数型，分别使用1、2、4、8个字节存储。
+ UInt8、UInt16、UInt32、UInt64、UInt128、UInt256：无符号整数型，分别使用1、2、4、8个字节存储。
+ Float32、Float64：浮点数型，分别使用4、8个字节存储。
    > 对浮点数进行计算可能引起四舍五入的误差。
    > SELECT 1 - 0.9  => 0.09999999999999998
+ Decimal(P, S)：定点数型，支持指定精度和小数位数。

#### 时间日期类型：

+ Date：日期类型，使用4个字节存储，表示自1970年1月1日以来的天数。
+ DateTime：日期时间类型，使用8个字节存储，精确到纳秒级。

#### 字符串类型：

+ FixedString(N)：定长字符串类型，占用N个字节存储。
+ String：变长字符串类型，使用相对较少的内存来存储字符串。

<!-- more -->

### 数据库引擎及基本语法

#### MergeTree

主键由关键字 PRIMARY KEY 指定，如果没有使用 PRIMARY KEY 显式指定的主键，ClickHouse 会使用排序键 ORDER BY 指定的列作为主键。

需要注意的是，如果同时指定了主键与排序键，那么排序键必须包含主键的所有列，比如主键为（a,b），排序键就必须为 (a,b,**)。

和关系型数据库的主键具备唯一性不同，**ClickHouse 可以存在相同主键的数据行**。这也是 ClickHouse 为了性能而做出的考量。

#### ReplacingMergeTree

前面提到 MergeTree 的主键没有唯一性约束。所以即使多行数据的主键相同，它们还是能够正常写入。

ReplacingMergeTree 建表语句与 MergeTree 相同，只需要替换掉表引擎就好，不过需要指定一个列作为版本列。

当排序键相同的数据行插入时，它会删除重复行，保留版本列值最大的一行。如果没有指定版本列，则会保留最后插入的一行。

通常来说，ReplacingMergeTree 删除重复行，是因为两种情况。

+ 一起批量插入的数据，属于同一个数据片段的数据，所以会在插入时候按排序键去重。
+ 分批插入的数据，属于不同的数据片段，需要依托后台数据片段合并的时候才进行。

所以即使你用了 ReplacingMergeTree 表引擎，在查询表数据的时候，仍然可能会查询到重复数据，不过只要使用 final 关键字去重即可。或者你也可以使用如下命令强制合并分区，这样数据片段就会合并。

### 四层数仓建模

在数仓建模中，有一个比较通用的建模方法论，就是将数据分为 4 层：包括：ODS（Operational Data Store）、DWD（Data Warehouse Detail）、DWS（Data Warehouse Summary）和 ADS（Application Data Store）

#### ODS

直接存放从各个渠道收集的原始数据，比如业务系统收集的、前后端埋点收集的等等。有时候会考虑做一定程度的数据清洗，比如处理异常字段、规范字段命名、统一时间字段等

#### DWD

以业务过程作为建模驱动，基于每个具体的业务过程特点，构建最细粒度的明细事实表，比如浏览明细表、交易明细表。DWD 也会清洗 ODS 层的数据（去除空值、脏数据、异常数据）、降维处理、脱敏等，并基于维度建模，将某些字段做适当冗余，做宽表化

#### DWS

在 DWD 层的基础上进行数据聚合和汇总，比如用户交易汇总表、用户指标汇总宽表。一般会依据特定业务需求或报表要求进行数据汇总和预计算，提高数据查询效率和性能

#### ADS

在 DWS 之上，面向特定应用场景设计的数据层，比如我们 CDP 场景里面，用户标签、人群画像，是通过计算后直接给到前端应用系统使用的表

### 搭建集群

ClickHouse 集群的配置很简单，只需要修改配置文件 /etc/clickhouse-server/config.xml，在配置文件的 标签下，增加集群的节点配置即可


#### remote_server 


配置集群时，主要不要配置 `<host>` 和 `<port>` 为转发端口。ck使用zookeeper作为分布式查询的同步节点，其中存储了分布式查询任务，可使用 zookeeper 客户端通过路径 `/clickhouse/task_queue/ddl/` 查看执行情况，查看其中任意任务内容即可发现，ck是根据配置中的 host 和 port 来明确需要执行分布式任务的服务

![avatar](ck1.png)

[stackoverflow](https://stackoverflow.com/questions/64947277/clickhouse-create-database-on-cluster-ends-with-timeout "ClickHouse的配置问题")  也有人反馈此问题
