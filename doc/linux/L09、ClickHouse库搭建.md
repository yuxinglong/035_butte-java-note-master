# 一、ClickHouse简介

## 1、基础简介

Yandex开源的数据分析的数据库，名字叫做ClickHouse，适合流式或批次入库的时序数据。ClickHouse不应该被用作通用数据库，而是作为超高性能的海量数据快速查询的分布式实时处理平台，在数据汇总查询方面(如GROUP BY)，ClickHouse的查询速度非常快。

```
下载仓库：https://repo.yandex.ru/clickhouse
中文文档：https://clickhouse.yandex/docs/zh/
```

## 2、数据库特点

(1)列式数据库

列式数据库是以列相关存储架构进行数据存储的数据库，主要适合于批量数据处理和即时查询。

(2)数据压缩

在一些列式数据库管理系统中不是用数据压缩。但是, 数据压缩在实现优异的存储系统中确实起着关键的作用。

(3)数据的磁盘存储

许多的列式数据库只能在内存中工作，这种方式会造成比实际更多的设备预算。ClickHouse被设计用于工作在传统磁盘上的系统，它提供每GB更低的存储成本。

(4)多核心并行处理

大型查询可以以很自然的方式在ClickHouse中进行并行化处理，以此来使用当前服务器上可用的所有资源。

(5)多服务器分布式处理

在ClickHouse中，数据可以保存在不同的shard上，每一个shard都由一组用于容错的replica组成，查询可以并行的在所有shard上进行处理。

(6)支持SQL和索引

ClickHouse支持基于SQL的查询语言，该语言大部分情况下是与SQL标准兼容的。支持的查询包括GROUPBY，ORDERBY，IN，JOIN以及非相关子查询。不支持窗口函数和相关子查询。按照主键对数据进行排序，这将帮助ClickHouse以几十毫秒的低延迟对数据进行特定值查找或范围查找。

(7)向量引擎

为了高效的使用CPU，数据不仅仅按列存储，同时还按向量(列的一部分)进行处理。

(8)实时的数据更新

ClickHouse支持在表中定义主键。为了使查询能够快速在主键中进行范围查找，数据总是以增量的方式有序的存储在MergeTree中。因此，数据可以持续不断高效的写入到表中，并且写入的过程中不会存在任何加锁的行为。

# 二、Linux下安装流程

**1、下载仓库**

```
curl -s 
https://packagecloud.io/install/repositories/altinity/clickhouse/script.rpm.sh 
| sudo os=centos dist=7 bash
```

**2、查看安装包**

```
sudo yum list 'clickhouse*'
```

**3、安装服务**

```
sudo yum install -y clickhouse-server clickhouse-client
```

**4、查看安装列表**

```
sudo yum list installed 'clickhouse*'
```

控制台输出

```
Installed Packages
clickhouse-client.noarch
clickhouse-common-static.x86_64
clickhouse-server.noarch
```

**5、查看配置**

- cd /etc/clickhouse-server/
- vim config.xml

```
数据目录：/var/lib/clickhouse/
临时目录：/var/lib/clickhouse/tmp/
日志目录：/var/log/clickhouse-server
HTTP端口：8123
TCP 端口：9000
```

**6、配置访问权限**

config.xml文件中去掉下面配置的注释。

```
<listen_host>::</listen_host> 
```

**7、启动服务**

```
/etc/rc.d/init.d/clickhouse-server start
```

**8、查看服务**

```
ps -aux |grep clickhouse
```

# 三、基础操作

## 1、建表语句

```
CREATE TABLE cs_user_info (
  `id` UInt64,
  `user_name` String,
  `pass_word` String,
  `phone` String,
  `email` String,
  `create_day` Date DEFAULT CAST(now(),'Date')
) ENGINE = MergeTree(create_day, intHash32(id), 8192)
```

注意事项：官方推荐引擎，MergeTree

Clickhouse 中最强大的表引擎当属MergeTree（合并树）引擎及该系列（*MergeTree）中的其他引擎。MergeTree引擎系列的基本理念如下。当你有巨量数据要插入到表中，你要高效地一批批写入数据片段，并希望这些数据片段在后台按照一定规则合并。相比在插入时不断修改（重写）数据进存储，这种策略会高效很多。

## 2、批量写入

```
INSERT INTO cs_user_info 
  (id,user_name,pass_word,phone,email) 
VALUES 
  (1,'cicada','123','13923456789','cicada@com'),
  (2,'smile','234','13922226789','smile@com'),
  (3,'spring','345','13966666789','spring@com');
```

## 3、查询语句

```
SELECT * FROM cs_user_info ;
SELECT * FROM cs_user_info WHERE user_name='smile' AND pass_word='234';
SELECT * FROM cs_user_info WHERE id IN (1,2);
SELECT * FROM cs_user_info WHERE id=1 OR id=2 OR id=3;
```

查询语句和操作MySQL数据库极其相似。

**源码参考：** https://gitee.com/cicadasmile/linux-system-base