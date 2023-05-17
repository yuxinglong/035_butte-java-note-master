# 一、Kafka集群环境

## 1、环境版本

```
版本：kafka2.11，zookeeper3.4
```

**注意**：这里zookeeper3.4也是基于集群模式部署。

## 2、解压重命名

```
tar -zxvf kafka_2.11-0.11.0.0.tgz
mv kafka_2.11-0.11.0.0 kafka2.11
```

创建日志目录

```
[root@en-master kafka2.11]# mkdir logs
```

**注意**：以上操作需要同步到集群下其他服务上。

## 3、添加环境变量

```
vim /etc/profile
export KAFKA_HOME=/opt/kafka2.11
export PATH=$PATH:$KAFKA_HOME/bin
source /etc/profile
```

## 4、修改核心配置

```
[root@en-master /opt/kafka2.11/config]# vim server.properties
-- 核心修改如下
# 唯一编号
broker.id=0
# 开启topic删除
delete.topic.enable=true
# 日志地址
log.dirs=/opt/kafka2.11/logs
# zk集群
zookeeper.connect=zk01:2181,zk02:2181,zk03:2181
```

**注意**：broker.id安装集群服务个数编排即可，集群下不能重复。

## 5、启动kafka集群

```
# 启动命令
[root@node02 kafka2.11]# bin/kafka-server-start.sh -daemon config/server.properties
# 停止命令
[root@node02 kafka2.11]# bin/kafka-server-stop.sh
# 进程查看
[root@node02 kafka2.11]# jps
```

**注意**：这里默认启动了zookeeper集群服务，并且集群下的kafka分别启动。

## 6、基础管理命令

**创建topic**

```
bin/kafka-topics.sh --zookeeper zk01:2181 \
--create --replication-factor 3 --partitions 1 --topic one-topic
```

参数说明：

- replication-factor  定义副本个数
- partitions  定义分区个数
- topic：定义topic名称

**查看topic列表**

```
bin/kafka-topics.sh --zookeeper zk01:2181 --list
```

**修改topic分区**

```
bin/kafka-topics.sh --zookeeper zk01:2181 --alter --topic one-topic --partitions 5
```

**查看topic**

```
bin/kafka-topics.sh --zookeeper zk01:2181 \
--describe --topic one-topic
```

**发送消息**

```
bin/kafka-console-producer.sh \
--broker-list 192.168.72.133:9092 --topic one-topic
```

**消费消息**

```
bin/kafka-console-consumer.sh \
--bootstrap-server 192.168.72.133:9092 --from-beginning --topic one-topic
```

**删除topic**

```
bin/kafka-topics.sh --zookeeper zk01:2181 \
--delete --topic first
```

## 7、Zk集群用处

Kafka集群中有一个broker会被选举为Controller，Controller依赖Zookeeper环境，管理集群broker的上下线，所有topic的分区副本分配和leader选举等工作。

# 二、消息拦截案例

## 1、拦截器简介

Kafka中间件的Producer拦截器主要用于实现消息发送的自定义控制逻辑。用户可以在消息发送前以及回调逻辑执行前有机会对消息做一些自定义，比如消息修改等，发送状态监控等，用户可以指定多个拦截器按顺序执行拦截。

**核心方法**

- configure：获取配置信息和初始化数据时调用；
- onSend：消息被序列化以及和计算分区前调用该方法，可以对消息做操作；
- onAcknowledgement：消息发送到Broker之后，或发送过程失败时调用；
- close：关闭拦截器调用，执行一些资源清理工作；

**注意**：这里说的拦截器是针对消息发送流程。

## 2、自定义拦截

**定义方式**：实现ProducerInterceptor接口即可。

**拦截器一**：在onSend方法中，对拦截的消息进行修改。

```java
@Component
public class SendStartInterceptor implements ProducerInterceptor<String, String> {

    private final Logger LOGGER = LoggerFactory.getLogger("SendStartInterceptor");
    @Override
    public void configure(Map<String, ?> configs) {
        LOGGER.info("configs...");
    }
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        // 修改消息内容
        return new ProducerRecord<>(record.topic(), record.partition(),
                                    record.timestamp(), record.key(),
                              "onSend：{" + record.value()+"}");
    }
    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        LOGGER.info("onAcknowledgement...");
    }
    @Override
    public void close() {
        LOGGER.info("SendStart close...");
    }
}
```

**拦截器二**：在onAcknowledgement方法中，判断消息是否发送成功。

```java
@Component
public class SendOverInterceptor implements ProducerInterceptor<String, String> {

    private final Logger LOGGER = LoggerFactory.getLogger("SendOverInterceptor");
    @Override
    public void configure(Map<String, ?> configs) {
        LOGGER.info("configs...");
    }

    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        LOGGER.info("record...{}", record.value());
        return record ;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception != null){
            LOGGER.info("Send Fail...exe-msg",exception.getMessage());
        }
        LOGGER.info("Send success...");
    }

    @Override
    public void close() {
        LOGGER.info("SendOver close...");
    }
}
```

**加载拦截器**：基于一个KafkaProducer配置Bean，加入拦截器。

```java
@Configuration
public class KafkaConfig {

    @Bean
    public Producer producer (){
        Properties props = new Properties();
        // 省略其他配置...
        // 添加拦截器
        List<String> interceptors = new ArrayList<>();
        interceptors.add("com.kafka.cluster.interceptor.SendStartInterceptor");
        interceptors.add("com.kafka.cluster.interceptor.SendOverInterceptor");
        props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptors);
        return new KafkaProducer<>(props) ;
    }
}
```

## 3、代码案例

```java
@RestController
public class SendMsgWeb {
    @Resource
    private KafkaProducer<String,String> producer ;
    @GetMapping("/sendMsg")
    public String sendMsg (){
        producer.send(new ProducerRecord<>("one-topic", "msgKey", "msgValue"));
        return "success" ;
    }
}
```

基于上述自定义Bean类型，进行消息发送，关注拦截器中打印日志信息。

# 三、Kafka存储分析

**说明**：该过程基于上述案例producer.send方法追踪的源码执行流程，源码中的过程相对清楚，涉及的核心流程如下。

## 1、消息生成过程

![](https://images.gitee.com/uploads/images/2022/0212/145959_29d138b3_5064118.png "08-1.png")

Producer发送消息采用的是异步发送的方式,消息发送过程如下：

- Producer发送消息之后，经过拦截器，序列化，事务判断；
- 流程执行后，消息内容放入容器中；
- 容器在指定时间内如果装满(size),会唤醒Sender线程;
- 容器如果在指定时间内没有装满，也会执行一次Sender线程唤醒；
- 唤醒Sender线程之后，把容器数据拉取到topic中；

**絮叨一句**：读这些中间件的源码，不仅能开阔思维，也会让自己意识到平时写的代码可能真的叫搬砖。

## 2、存储机制

Kafka中消息是以topic进行标识分类，生产者面向topic生产消息，topic分区(partition)是物理上的存储，基于消息日志文件的方式。

![](https://images.gitee.com/uploads/images/2022/0212/150011_7ad26055_5064118.jpeg "08-2.jpg")

- 每个partition对应于一个log文件,发送的消息不断追加到该log文件末端；
- log文件中存储的就是producer生产的消息数据，采用分片和索引机制；
- partition分为多个segment。每个segment对应两个(.index)和(.log)文件；
- index文件类型存储的索引信息；
- log文件存储消息的数据；
- 索引文件中的元数据指向对应数据文件中message的物理偏移地址；
- 消费者组中的每个消费者，都会实时记录消费的消息offset位置；
- 当然消息消费出错时，恢复是从上次的记录位置继续消费；

## 3、事务控制机制

![](https://images.gitee.com/uploads/images/2022/0212/150023_0456bd07_5064118.jpeg "08-3.jpg")

Kafka支持消息的事务控制

**Producer事务**

跨分区跨会话的事务原理，引入全局唯一的TransactionID，并将Producer获得的PID和TransactionID绑定。Producer重启后可以通过正在进行的TransactionID获得原来的PID。
Kafka基于TransactionCoordinator组件管理Transaction，Producer通过和TransactionCoordinator交互获得TransactionID对应的任务状态。TransactionCoordinator将事务状态写入Kafka的内部Topic，即使整个服务重启，进行中的事务状态可以得到恢复。

**Consumer事务**

Consumer消息消费，事务的保证强度很低，无法保证消息被精确消费，因为同一事务的消息可能会出现重启后已经被删除的情况。

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap01-data-source/case08-kafka-cluster