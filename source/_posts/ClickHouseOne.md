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

{% note danger %}

`PARTITION BY` 分区键，可选项。大多数情况下，不需要使用分区键。即使需要使用，也不需要使用比月更细粒度的分区键。分区不会加快查询（这与 ORDER BY 表达式不同）。永远也别使用过细粒度的分区键。不要使用客户端指定分区标识符或分区字段名称来对数据进行分区（而是将分区字段标识或名称作为 ORDER BY 表达式的第一列来指定分区）。

要按月分区，可以使用表达式 toYYYYMM(date_column) ，这里的 date_column 是一个 Date 类型的列。分区名的格式会是 "YYYYMM" 。

{% endnote %}

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

配置集群时，注意不要配置 `<host>` 和 `<port>` 为转发端口。ck使用zookeeper作为分布式查询的同步节点，其中存储了分布式查询任务，可使用 zookeeper 客户端通过路径 `/clickhouse/task_queue/ddl/` 查看执行情况，查看其中任意任务内容即可发现，ck是根据配置中的 host 和 port 来明确需要执行分布式任务的服务

![avatar](ck1.png)

[stackoverflow](https://stackoverflow.com/questions/64947277/clickhouse-create-database-on-cluster-ends-with-timeout "ClickHouse的配置问题")  也有人反馈此问题

### 常用命令

#### 查看进程及杀死进程

当出现 ClickHouse 查询过慢，所在服务器cpu和内存飙升的情况，说明可能出现慢 sql，此时可通过以下命令查询正在执行的 sql 信息

``` sql
select
query_id,read_rows,total_rows_approx,memory_usage,
initial_user,initial_address,elapsed,query
from system.processes;
```

字段含义
- query_id 查询id
- read_rows 从表中读取的行数
- total_rows_approx 应读取的行总数的近似值
- memory_usage 请求使用的内存量
- initial_user 进行查询的用户
- initial_address 请求的 IP 地址
- elapsed 从执行开始以来的秒数
- query 查询语句

通过sql语句的查询行数和查询已经执行的时间来判断sql是不是在慢查询，或者是同事在查询的时候没有日期限定而直接查全表。一般的话如果grafana监控的CK节点出现cpu飙升的情况，就需要我们去判断CK中是否有垃圾sql在执行，根据query_id杀死该进程

``` sql
kill query where query_id = '70442d9b-7fc5-4a0e-81be-9543431a4882' ;
```

#### 清表

如果发现数据集错误，可以执行下述命令清除表中数据，但是执行较慢，谨慎执行

``` sql
TRUNCATE TABLE cdp.cdp_user_local;
```

#### 分批导入数据

可用官方客户端工具 clickhouse-client 导入数据，参考命令

``` shell
clickhouse-client --password 123456 --port 9000 --query="INSERT INTO cdp.cdp_user_all FORMAT CSV" --input_format_allow_errors_ratio=0.1 --input_format_allow_errors_num=0 < cdp_user_data.csv
```

在 ClickHouse 里，其实不建议表分区数量太多。默认情况下一次插入数据的分区超过 100 个，就会报下面的错误。

``` shell
Received exception from server (version 24.3.2):
Code: 252. DB::Exception: Received from localhost:9002. DB::Exception: Too many partitions for single INSERT block (more than 100). The limit is controlled by 'max_partitions_per_insert_block' setting. Large number of partitions is a common misconception. It will lead to severe negative performance impact, including slow server startup, slow INSERT queries and slow SELECT queries. Recommended total number of partitions for a table is under 1000..10000. Please note, that partitioning is not intended to speed up SELECT queries (ORDER BY key is sufficient to make range queries fast). Partitions are intended for data manipulation (DROP PARTITION, etc).. (TOO_MANY_PARTS)
(query: INSERT INTO cdp.cdp_user_all FORMAT CSV)
```

可以通过修改 /etc/clickhouse-server/users.xml 配置文件的 max_partitions_per_insert_block 的值来调大每次 INSERT 可以承载的分区数量。

``` xml
<profiles>
    <!-- Default settings. -->
    <default>
        <max_partitions_per_insert_block>10000</max_partitions_per_insert_block>
    </default>
</profiles>
```

还有另一种方式，拆解要插入的csv文件，可在 linux 下执行如下命令

``` shell
split -l 1000 cdp_user_data.csv output_chunk_
```

这里，`-l 1000` 参数指定每个生成的文件应该包含1000行，`cdp_user_data.csv` 是要拆分的原始文件，`output_chunk_` 是每个小文件的前缀。执行上述命令后，你会得到类似 `output_chunk_aa`, `output_chunk_ab`, `output_chunk_ac` 等以 `output_chunk_` 开头的文件，每个文件包含大约1000行（最后一个文件可能少于1000行，如果原始文件的行数不是1000的倍数）。

随后执行如下命令即可批量将拆分后的文件导入到 ClickHouse

``` shell
cat *.csv | clickhouse-client --password 123456 --port 9000 --query="INSERT INTO cdp.cdp_user_all FORMAT CSV" --input_format_allow_errors_ratio=0.1 --input_format_allow_errors_num=0
```

### 踩坑问题

#### 分区数量过多

数据测试时，错误使用了 `PARTITION BY`，选择了一个区分度过高，类似于主键的字段作为分区键，导致每插入一条数据就会创建一个分区，当插入20w数据时，就达到分区上限（搭建了两个节点组建集群，单个节点默认上限10w个分区），建表语句如下

``` sql
CREATE TABLE cdp_user_local on cluster default(
  unique_user_id UInt64 COMMENT '用户全局唯一ID，ONE-ID',
  name String COMMENT '用户姓名',
  nickname String COMMENT '用户昵称',
  gender Int8 COMMENT '性别：1-男；2-女；3-未知',
  birthday String COMMENT '用户生日：yyyyMMdd',
  user_level Int8 COMMENT '用户等级：1-5',
  register_date DateTime64 COMMENT '注册日期',
  last_login_time DateTime64 COMMENT '最后一次登录时间'
) ENGINE = MergeTree()
    PARTITION BY register_date
    PRIMARY KEY unique_user_id
    ORDER BY unique_user_id;
```

register_date 时间格式为年月日时分秒，区分度过高，错误如下。此时应参考官方建议使用函数 `toYYYYMM(register_date)` 

``` shell
Code: 252. DB::Exception: Too many parts (100025) in all partitions in total. This indicates wrong choice of partition key. The threshold can be modified with 'max_parts_in_total' setting in <merge_tree> element in config.xml or with per-table setting. (TOO_MANY_PARTS) (version 21.11.11.1 (official build))
```

正常情况下不需要按照报错提示修改分区上限，大部分情况是因为错误的设置了细粒度分区键导致的，请自行检查建表 `sql`，如需要修改，可编辑 `/etc/clickhouse-server/users.xml` 文件，添加如下配置项。

``` xml
<merge_tree>
    <max_parts_in_total>100000</max_parts_in_total>
</merge_tree>
```

#### datagrip查询超时

有时会执行一些时间较长的慢 `sql`，但是在 datagrip 软件下，默认30s会超时提示 `Read timed out`，datagrip 默认使用 jdbc 链接，可手动修改配置项

![avatar](ck2.png)

亦可直接在链接上添加参数 `jdbc:clickhouse://localhost:8123?socket_timeout=300000`

