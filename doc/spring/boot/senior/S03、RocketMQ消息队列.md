
# 一、RocketMQ

## 1、架构图片

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/135633_8cae3ba5_5064118.png "01-1.png")

## 2、角色分类

(1)、Broker

RocketMQ 的核心，接收 Producer 发过来的消息、处理 Consumer 的消费消息请求、消息的持 久化存储、服务端过滤功能等 。

(2)、NameServer

消息队列中的状态服务器，集群的各个组件通过它来了解全局的信息 。类似微服务中注册中心的服务注册，发现，下线，上线的概念。

热备份：

NamServer可以部署多个，相互之间独立，其他角色同时向多个NameServer 机器上报状态信息。

心跳机制：

NameServer 中的 Broker、 Topic等状态信息不会持久存储，都是由各个角色定时上报并存储到内存中，超时不上报的话， NameServer会认为某个机器出故障不可用。

(3)、Producer

消息的生成者，最常用的producer类就是DefaultMQProducer。

(4)、Consumer

消息的消费者，常用Consumer类

- DefaultMQPushConsumer：收到消息后自动调用传入的处理方法来处理，实时性高
- DefaultMQPullConsumer：用户自主控制 ，灵活性更高。

## 3、通信机制

(1)、Broker启动后需要完成一次将自己注册至NameServer的操作；随后每隔30s时间定时向NameServer更新Topic路由信息。

(2)、Producer发送消息时候，需要根据消息的Topic从本地缓存的获取路由信息。如果没有则更新路由信息会从NameServer重新拉取，同时Producer会默认每隔30s向NameServer拉取一次路由信息。

(3)、Consumer消费消息时候，从NameServer获取的路由信息，并再完成客户端的负载均衡后，监听指定消息队列获取消息并进行消费。

# 二、代码实现案例

## 1、项目结构图

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/135732_863c9df8_5064118.png "01-2.png")

版本描述

```
<spring-boot.version>2.1.3.RELEASE</spring-boot.version>
<rocketmq.version>4.3.0</rocketmq.version>
```

## 2、配置文件

```yaml
rocketmq:
  # 生产者配置
  producer:
    isOnOff: on
    # 发送同一类消息的设置为同一个group，保证唯一
    groupName: CicadaGroup
    # 服务地址
    namesrvAddr: 127.0.0.1:9876
    # 消息最大长度 默认1024*4(4M)
    maxMessageSize: 4096
    # 发送消息超时时间,默认3000
    sendMsgTimeout: 3000
    # 发送消息失败重试次数，默认2
    retryTimesWhenSendFailed: 2
  # 消费者配置
  consumer:
    isOnOff: on
    # 官方建议：确保同一组中的每个消费者订阅相同的主题。
    groupName: CicadaGroup
    # 服务地址
    namesrvAddr: 127.0.0.1:9876
    # 接收该 Topic 下所有 Tag
    topics: CicadaTopic~*;
    consumeThreadMin: 20
    consumeThreadMax: 64
    # 设置一次消费消息的条数，默认为1条
    consumeMessageBatchMaxSize: 1

# 配置 Group  Topic  Tag
rocket:
  group: rocketGroup
  topic: rocketTopic
  tag: rocketTag
```

## 3、生产者配置

``` java
/**
 * RocketMQ 生产者配置
 */
@Configuration
public class ProducerConfig {

    private static final Logger LOG = LoggerFactory.getLogger(ProducerConfig.class) ;
    @Value("${rocketmq.producer.groupName}")
    private String groupName;
    @Value("${rocketmq.producer.namesrvAddr}")
    private String namesrvAddr;
    @Value("${rocketmq.producer.maxMessageSize}")
    private Integer maxMessageSize ;
    @Value("${rocketmq.producer.sendMsgTimeout}")
    private Integer sendMsgTimeout;
    @Value("${rocketmq.producer.retryTimesWhenSendFailed}")
    private Integer retryTimesWhenSendFailed;

    @Bean
    public DefaultMQProducer getRocketMQProducer() {
        DefaultMQProducer producer;
        producer = new DefaultMQProducer(this.groupName);
        producer.setNamesrvAddr(this.namesrvAddr);
        //如果需要同一个jvm中不同的producer往不同的mq集群发送消息，需要设置不同的instanceName
        if(this.maxMessageSize!=null){
            producer.setMaxMessageSize(this.maxMessageSize);
        }
        if(this.sendMsgTimeout!=null){
            producer.setSendMsgTimeout(this.sendMsgTimeout);
        }
        //如果发送消息失败，设置重试次数，默认为2次
        if(this.retryTimesWhenSendFailed!=null){
            producer.setRetryTimesWhenSendFailed(this.retryTimesWhenSendFailed);
        }
        try {
            producer.start();
        } catch (MQClientException e) {
            e.printStackTrace();
        }
        return producer;
    }
}
```

## 4、消费者配置

```java
/**
 * RocketMQ 消费者配置
 */
@Configuration
public class ConsumerConfig {
    private static final Logger LOG = LoggerFactory.getLogger(ConsumerConfig.class) ;
    @Value("${rocketmq.consumer.namesrvAddr}")
    private String namesrvAddr;
    @Value("${rocketmq.consumer.groupName}")
    private String groupName;
    @Value("${rocketmq.consumer.consumeThreadMin}")
    private int consumeThreadMin;
    @Value("${rocketmq.consumer.consumeThreadMax}")
    private int consumeThreadMax;
    @Value("${rocketmq.consumer.topics}")
    private String topics;
    @Value("${rocketmq.consumer.consumeMessageBatchMaxSize}")
    private int consumeMessageBatchMaxSize;
    @Resource
    private RocketMsgListener msgListener;
    @Bean
    public DefaultMQPushConsumer getRocketMQConsumer(){
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(groupName);
        consumer.setNamesrvAddr(namesrvAddr);
        consumer.setConsumeThreadMin(consumeThreadMin);
        consumer.setConsumeThreadMax(consumeThreadMax);
        consumer.registerMessageListener(msgListener);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        consumer.setConsumeMessageBatchMaxSize(consumeMessageBatchMaxSize);
        try {
            String[] topicTagsArr = topics.split(";");
            for (String topicTags : topicTagsArr) {
                String[] topicTag = topicTags.split("~");
                consumer.subscribe(topicTag[0],topicTag[1]);
            }
            consumer.start();
        }catch (MQClientException e){
            e.printStackTrace();
        }
        return consumer;
    }
}
```

## 5、消息监听配置

```java
/**
 * 消息消费监听
 */
@Component
public class RocketMsgListener implements MessageListenerConcurrently {
    private static final Logger LOG = LoggerFactory.getLogger(RocketMsgListener.class) ;
    @Resource
    private ParamConfigService paramConfigService ;
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext context) {
        if (CollectionUtils.isEmpty(list)){
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
        MessageExt messageExt = list.get(0);
        LOG.info("接受到的消息为："+new String(messageExt.getBody()));
        int reConsume = messageExt.getReconsumeTimes();
        // 消息已经重试了3次，如果不需要再次消费，则返回成功
        if(reConsume ==3){
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
        if(messageExt.getTopic().equals(paramConfigService.rocketTopic)){
            String tags = messageExt.getTags() ;
            switch (tags){
                case "rocketTag":
                    LOG.info("开户 tag == >>"+tags);
                    break ;
                default:
                    LOG.info("未匹配到Tag == >>"+tags);
                    break;
            }
        }
        // 消息消费成功
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}
```

## 6、配置参数绑定

```java
@Service
public class ParamConfigService {
    @Value("${rocket.group}")
    public String rocketGroup ;
    @Value("${rocket.topic}")
    public String rocketTopic ;
    @Value("${rocket.tag}")
    public String rocketTag ;
}
```

## 7、消息发送测试

```java
@Service
public class RocketMqServiceImpl implements RocketMqService {
    @Resource
    private DefaultMQProducer defaultMQProducer;
    @Resource
    private ParamConfigService paramConfigService ;
    @Override
    public SendResult openAccountMsg(String msgInfo) {
        // 可以不使用Config中的Group
        defaultMQProducer.setProducerGroup(paramConfigService.rocketGroup);
        SendResult sendResult = null;
        try {
            Message sendMsg = new Message(paramConfigService.rocketTopic,
                                          paramConfigService.rocketTag,
                                         "open_account_key", msgInfo.getBytes());
            sendResult = defaultMQProducer.send(sendMsg);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return sendResult ;
    }
}
```

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware03-rocket-queue