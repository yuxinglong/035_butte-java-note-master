
# 一、Seata简介

## 1、Seata组件

Seata是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata将为用户提供了AT、TCC、SAGA、XA事务模式，为用户打造一站式的分布式解决方案。

## 2、支持模式

**AT 模式**

- 基于支持本地 ACID 事务的关系型数据库。
- Java应用，通过 JDBC 访问数据库。

一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。

二阶段：提交异步化，非常快速地完成。回滚通过一阶段的回滚日志进行反向补偿。

**TCC模式**

一个分布式的全局事务，整体是两阶段提交的模型，全局事务是由若干分支事务组成的，分支事务要满足两阶段提交的模型要求，即需要每个分支事务都具备自己的：

一阶段 prepare 行为

二阶段 commit 或 rollback 行为

**Saga模式**

Saga模式是SEATA提供的长事务解决方案，在Saga模式中，业务流程中每个参与者都提交本地事务，当出现某一个参与者失败则补偿前面已经成功的参与者，一阶段正向服务和二阶段补偿服务都由业务开发实现。

**XA模式**

XA是一个分布式事务协议，对业务无侵入的分布式事务解决方案，XA提交协议需要事务参与者的数据库支持，XA事务具有强一致性，在两阶段提交的整个过程中，一直会持有资源的锁，性能不理想的缺点很明显。

# 二、服务端部署

## 1、下载组件包

1.2版本：seata-server-1.2.0.zip

解压目录

- bin：存放服务端运行启动脚本; 
- lib：存放服务端依赖的资源jar包；
- conf：配置文件目录。

## 2、修改配置

**file.conf配置**

mode:db 即使用数据库存储事务信息，这里还可以选择file存储方式。

file模式为单机模式，全局事务会话信息内存中读写并持久化本地文件root.data，性能较高;

db模式为高可用模式，全局事务会话信息通过db共享，相应性能差些;

redis模式Seata-Server 1.3及以上版本支持,性能较高,存在事务信息丢失风险,请提前配置合适当前场景的redis持久化配置.

```
store {
  ## store mode: file、db
  mode = "db"
  db {
    datasource = "druid"
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata_server"
    user = "root"
    password = "123456"
    minConn = 5
    maxConn = 30
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

**registry.conf配置**

这里选择eureka作为注册中心，seata-server也要作为一个服务添加到注册中心，不使用配置中心所以config配置默认即可。

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "eureka"

  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
}
```

## 3、事务管理表

需要在seata-server即上述配置的MySQL库中建立3张事务管理表：

- 全局事务：global_table
- 分支事务：branch_table
- 全局锁：lock_table
- 事务回滚：undo_log
- SQL脚本：mysql-script目录

## 4、启动命令

Linux环境：sh seata-server.sh

# 三、业务服务搭建

## 1、代码结构

![](https://images.gitee.com/uploads/images/2022/0208/223933_5eb0f847_5064118.png "12-1.png")

- seata-eureka：注册中心
- seata-order：订单服务
- seata-account：账户服务
- seata-inventor：库存服务
- seata-client：客户端服务
- account-feign：账户Feign接口
- inventory-feign：库存Feign接口
- order-feign：订单Feign接口

**请求链路**：客户端->订单->账户+库存，测试整个流程的分布式事务问题。

## 2、数据库结构

![](https://images.gitee.com/uploads/images/2022/0208/223957_2d794f6f_5064118.png "12-2.png")

- seata_server：seata组件服务端依赖库
- seata_account：模拟账户数据库
- seata_inventor：模拟库存数据库
- seata_order：模拟订单数据库

各个库脚本位置：mysql-script/data-biz.sql

## 3、启动服务

依次启动：注册中心，库存服务，账户服务，订单服务，客户端服务；

Eureka服务列表如下：

![](https://images.gitee.com/uploads/images/2022/0208/224009_28818c8b_5064118.png "12-3.png")

# 四、Seata用法详解

## 1、Seata基础配置

![](https://images.gitee.com/uploads/images/2022/0208/224022_d114592e_5064118.png "12-4.png")

几个基础服务的配置方式一样。

**conf配置**

file.conf重点关注下面内容，事务组的名称，需要在yml文件中使用。

```
my_test_tx_group = "default"
```

**registry.conf**：是注册中心的选择。

## 2、数据库配置

注意这里的事务组名称配置。

```
spring:
  # 事务组的名称
  cloud:
    alibaba:
      seata:
        tx-service-group: my_test_tx_group
  # 数据源配置
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driverClassName: com.mysql.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/seata_account
      username: root
      password: 123456
```

将数据库整体由Seata进行代理管理，核心API：DataSourceProxy。

```java
@Configuration
public class SeataAccountConfig {

    @Value("${spring.application.name}")
    private String applicationName;

    @Bean
    public GlobalTransactionScanner globalTransactionScanner() {
        return new GlobalTransactionScanner(applicationName, "test-tx-group");
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid")
    public DruidDataSource druidDataSource() {
        return new DruidDataSource() ;
    }

    @Primary
    @Bean("dataSource")
    public DataSourceProxy dataSourceProxy(DataSource druidDataSource) {
        return new DataSourceProxy(druidDataSource);
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSourceProxy dataSourceProxy)throws Exception{
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSourceProxy);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath*:/mapper/*.xml"));
        sqlSessionFactoryBean.setTransactionFactory(new SpringManagedTransactionFactory());
        return sqlSessionFactoryBean.getObject();
    }
}
```

## 3、业务代码

核心注解：GlobalTransactional，管理整体的分布式事务。

```
@Service
public class OrderServiceImpl implements OrderService {
    private final Logger LOGGER = LoggerFactory.getLogger(OrderServiceImpl.class);

    @Resource
    private OrderMapper orderMapper ;
    @Resource
    private AccountFeign accountFeign ;
    @Resource
    private InventoryFeign inventoryFeign ;

    @GlobalTransactional
    @Override
    public Integer createOrder(String orderNo) {
        LOGGER.info("Order 生成中 "+orderNo);
        // 本服务下订单库
        Integer insertFlag = orderMapper.insert(orderNo) ;
        // 基于feign接口处理账户和库存
        accountFeign.updateAccount(10L) ;
        inventoryFeign.updateInventory(10) ;
        return insertFlag ;
    }
}
```

测试流程：在任意服务下抛出异常，观察整体的事务状态，观察是否有整体的事务控制效果。

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node11-parent