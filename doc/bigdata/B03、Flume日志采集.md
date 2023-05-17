# 一、Flume简介

## 1、基础描述

Flume是Cloudera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume支持在日志系统中定制各类数据发送方，用于收集数据；

特点：分布式、高可用、基于流式架构，通常用来收集、聚合、搬运不同数据源的大量日志到数据仓库。

## 2、架构模型

![](https://images.gitee.com/uploads/images/2022/0213/141900_22a71fce_5064118.png "01-1.png")

Agent包括三个核心组成，Source、Channel、Sink。Source负责接收数据源，并兼容多种类型，Channel是数据的缓冲区，Sink处理数据输出的方式和目的地。

Event是Flume定义的一个数据流传输的基本单元，将数据从源头送至目的地。

![](https://images.gitee.com/uploads/images/2022/0213/141914_85f484df_5064118.png "01-2.png")

Flume可以设置多级Agent连接的方式传输Event数据，从最初的source开始到最终sink传送的目的存储系统，如果数量过多会影响传输速率，并且传输过程中单节点故障也会影响整个传输通道。

![](https://images.gitee.com/uploads/images/2022/0213/141926_31e41d3a_5064118.png "01-3.png")

Flume支持多路复用数据流到一个或多个目的地，这种模式可以将相同数据复制到多个channel中，或者将不同数据分发到不同的channel中，并且sink可以选择传送到不同的目的地。

![](https://images.gitee.com/uploads/images/2022/0213/141939_5854d267_5064118.png "01-4.png")

Agent1理解为路由节点负责Channel的Event均衡到多个Sink组件，每个Sink组件分別连接到独立的Agent上，实现负载均衡和错误恢复的功能。

![](https://images.gitee.com/uploads/images/2022/0213/141951_5ffa3d30_5064118.png "01-5.png")

Flume的使用组合方式做数据聚合，每台服务器部署一个flume节点采集日志数据，再汇聚传输到存储系统，例如HDFS、Hbase等组件，高效且稳定的解决集群数据的采集。

# 二、安装过程

## 1、安装包

apache-flume-1.7.0-bin.tar.gz

## 2、解压命名

```
[root@hop01 opt]# pwd
/opt
[root@hop01 opt]# tar -zxf apache-flume-1.7.0-bin.tar.gz
[root@hop01 opt]# mv apache-flume-1.7.0-bin flume1.7
```

## 3、配置文件

配置路径：/opt/flume1.7/conf

```
mv flume-env.sh.template flume-env.sh
```

## 4、修改配置

添加JDK依赖

```
vim flume-env.sh
export JAVA_HOME=/opt/jdk1.8
```

## 5、环境测试

安装netcat工具

```
sudo yum install -y nc
```

创建任务配置

```
[root@hop01 flume1.7]# cd job/
[root@hop01 job]# vim flume-netcat-test01.conf
```

添加基础任务配置

注意：a1表示agent名称。
```
# this agent
a1.sources = sr1
a1.sinks = sk1
a1.channels = sc1

# the source
a1.sources.sr1.type = netcat
a1.sources.sr1.bind = localhost
a1.sources.sr1.port = 55555

# the sink
a1.sinks.sk1.type = logger

# events in memory
a1.channels.sc1.type = memory
a1.channels.sc1.capacity = 1000
a1.channels.sc1.transactionCapacity = 100

# Bind the source and sink
a1.sources.sr1.channels = sc1
a1.sinks.sk1.channel = sc1
```

开启flume监听端口

```
/opt/flume1.7/bin/flume-ng agent --conf /opt/flume1.7/conf/ --name a1 --conf-file /opt/flume1.7/job/flume-netcat-test01.conf -Dflume.root.logger=INFO,console
```

使用netcat工具向55555端口发送数据

```
[root@hop01 ~]# nc localhost 55555
hello,flume
```

查看flume控制面

![](https://images.gitee.com/uploads/images/2022/0213/142030_0172f274_5064118.png "01-6.png")

# 三、应用案例

## 1、案例描述

![](https://images.gitee.com/uploads/images/2022/0213/142038_c0a9bb8d_5064118.png "01-7.png")

基于flume在各个集群服务进行数据采集，然后数据传到kafka服务，再考虑数据的消费策略。

采集：基于flume组件的便捷采集能力，如果直接使用kafka会产生大量的埋点动作不好维护。

消费：基于kafka容器的数据临时存储能力，避免系统高度活跃期间采集数据过大冲垮数据采集通道，并且可以基于kafka做数据隔离并针对化处理。

## 2、创建kafka配置

```
[root@hop01 job]# pwd
/opt/flume1.7/job
[root@hop01 job]# vim kafka-flume-test01.conf
```

## 3、修改sink配置

```
# the sink
a1.sinks.sk1.type = org.apache.flume.sink.kafka.KafkaSink
# topic
a1.sinks.sk1.topic = kafkatest
# broker地址、端口号
a1.sinks.sk1.kafka.bootstrap.servers = hop01:9092
# 序列化方式
a1.sinks.sk1.serializer.class = kafka.serializer.StringEncoder
```

## 4、创建kafka的Topic

上述配置文件中名称：kafkatest，下面执行创建命令之后查看topic信息。

```
[root@hop01 bin]# pwd
/opt/kafka2.11
[root@hop01 kafka2.11]# bin/kafka-topics.sh --create --zookeeper hop01:2181 --replication-factor 1 --partitions 1 --topic kafkatest
[root@hop01 kafka2.11]# bin/kafka-topics.sh --describe --zookeeper hop01:2181 --topic kafkatest
```

## 5、启动Kakfa消费

```
[root@hop01 kafka2.11]# bin/kafka-console-consumer.sh --bootstrap-server hop01:2181 --topic kafkatest --from-beginning
```

这里指定topic是kafkatest。

## 6、启动flume配置

```
/opt/flume1.7/bin/flume-ng agent --conf /opt/flume1.7/conf/ --name a1 --conf-file /opt/flume1.7/job/kafka-flume-test01.conf -Dflume.root.logger=INFO,console
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent