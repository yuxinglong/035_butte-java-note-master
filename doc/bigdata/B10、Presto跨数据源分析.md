# 一、Presto概述

## 1、Presto简介

Presto是一个开源的分布式SQL查询引擎，适用于交互式分析查询，数据量支持GB到PB字节，Presto虽然具备解析SQL的能力，但它并不属于标准的数据库范畴。

Presto支持在线数据查询，包括Hive，关系数据库以及专有数据存储。一条Presto查询可以将多个数据源的数据进行合并，可以跨越整个组织进行分析，Presto主要用来处理响应时间小于1秒到几分钟的场景。

## 2、Presto架构

Presto查询引擎是基于Master-Slave的架构，运行在多台服务器上的分布式系统，由一个Coordinator节点和多个Worker节点组成，Coordinator负责解析SQL语句，生成执行计划，分发执行任务给Worker节点执行，Worker节点负责实际执行查询任务。

![](https://images.gitee.com/uploads/images/2022/0213/152547_0136a03e_5064118.png "01-1.png")

**Coordinator节点**

Coordinator服务器是用来解析查询语句，执行计划分析和管理Presto的Worker结点，跟踪每个Work的活动情况并协调查询语句的执行。Coordinator为每个查询建立模型，模型包含多个Stage，每个Stage再转为Task分发到不同的Worker上执行，协调通信基于REST-API，Presto安装必须有一个Coordinator节点。

**Worker节点**

Worker负责执行查询任务和处理数据，从Connector获取数据，Worker间会交换中间数据。Coordinator从Worker获取结果并返回最终结果给Client端，当Worker启动时会广播自己并发现Coordinator，告知Coordinator可用状态，协调通信基于REST-API，Presto通常会安装多个Worker节点。

**数据源适配**

Presto可以适配多种不同的数据源，可以和数据源连接和交互，Presto是通过表的完全限定名处理table，Catalog对应类数据源，Schema对应数据库，Table对应数据表。

![](https://images.gitee.com/uploads/images/2022/0213/152613_2ae5f074_5064118.png "01-2.png")

Presto中处理的最小数据单元是一个Page对象，一个Page对象包含多个Block对象，每个Block对象是一个字节数组，存储一个字段的若干行，多个Block横切的一行是真实的一行数据。

# 二、Presto安装

## 1、安装包管理

```
[root@hop01 presto]# pwd
/opt/presto
[root@hop01 presto]# ll
presto-cli-0.196-executable.jar
presto-server-0.189.tar.gz
[root@hop01 presto]# tar -zxvf presto-server-0.189.tar.gz
```

## 2、配置管理

在presto安装目录中创建etc文件夹，并添加以下配置信息：

```
/opt/presto/presto-server-0.189/etc
```

**节点属性**

每个节点的特定环境配置:etc/node.properties；

```
[root@hop01 etc]# vim node.properties
node.environment=production
node.id=presto01
node.data-dir=/opt/presto/data
```

配置内容：环境名称，唯一ID，数据目录。

**JVM 配置**

JVM的命令行选项，用于启动Java虚拟机的命令行选项列表:etc/jvm.config。

```
[root@hop01 etc]# vim jvm.config
-server
-Xmx16G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
```

**配置属性**

Presto服务器的配置，每个Presto服务器都可以充当协调器和工作器，如果单独使用一台机器来执行协调工作可以在更大的集群上提供最佳性能，这里PrestoServer既当一个coordinator也是一个worker节点:etc/config.properties。

```
[root@hop01 etc]# vim config.properties
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8083
query.max-memory=3GB
query.max-memory-per-node=1GB
discovery-server.enabled=true
discovery.uri=http://hop01:8083
```

这里coordinator=true表示当前Presto实例充当协调器角色。

**日志配置**

```
[root@hop01 etc]# vim log.properties
com.facebook.presto=INFO
```

**Catalog属性**

```
/opt/presto/presto-server-0.189/etc/catalog
```

配置hive适配:

```
[root@hop01 catalog]# vim hive.properties
connector.name=hive-hadoop2
hive.metastore.uri=thrift://192.168.37.133:9083
```

配置MySQL适配:

```
[root@hop01 catalog]# vim mysql.properties
connector.name=mysql
connection-url=jdbc:mysql://192.168.37.133:3306
connection-user=root
connection-password=123456
```

![](https://images.gitee.com/uploads/images/2022/0213/152649_7fd0c8ed_5064118.png "01-3.png")

## 3、运行服务

启动命令

```
[root@hop01 /]# /opt/presto/presto-server-0.189/bin/launcher run
```

启动日志

![](https://images.gitee.com/uploads/images/2022/0213/152704_abe7bc58_5064118.png "01-4.png")

这样presto就启动成功了。

# 三、客户端安装

## 1、Jar包管理

```
[root@hop01 presto-cli]# pwd
/opt/presto/presto-cli
[root@hop01 presto-cli]# ll
presto-cli-0.196-executable.jar
[root@hop01 presto-cli]# mv presto-cli-0.196-executable.jar presto-cli.jar
```

## 2、连接MySQL

![](https://images.gitee.com/uploads/images/2022/0213/152716_8643fbe4_5064118.png "01-5.png")

```
java -jar presto-cli.jar --server ip:9000 --catalog mysql --schema sq_export
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent