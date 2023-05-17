
# 一、AliCloud简介

# 1、基础描述

Alibaba-Cloud致力于提供微服务开发的一站式解决方案。此项目包含开发分布式应用微服务的必需组件，方便开发者通过SpringCloud编程模型轻松使用这些组件来开发分布式应用服务。只需要添加一些注解和少量配置，就可以将SpringCloud应用接入阿里微服务解决方案，通过阿里中间件来迅速搭建分布式应用系统。

# 2、核心功能

- 服务限流降级

默认支持 WebServlet、WebFlux, OpenFeign、RestTemplate、SpringCloudGateway,Zuul,Dubbo和RocketMQ限流降级功能的接入，可以在运行时通过控制台实时修改限流降级规则，还支持查看限流降级 Metrics 监控。

- 服务注册与发现

适配 Spring Cloud 服务注册与发现标准，默认集成了 Ribbon 的支持。

- 分布式配置管理

支持分布式系统中的外部化配置，配置更改时自动刷新。

- 消息驱动能力

基于 Spring Cloud Stream 为微服务应用构建消息驱动能力。

- 分布式事务

使用 @GlobalTransactional 注解， 高效并且对业务零侵入地解决分布式事务问题。

- 分布式任务调度

提供秒级、精准、高可靠、高可用的定时（基于Cron表达式）任务调度服务。同时提供分布式的任务执行模型，如网格任务。网格任务支持海量子任务均匀分配到所有 Worker（schedulerx-client）上执行。

## 3、功能组件

- Nacos

一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

- Sentinel

把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。

- RocketMQ

一款开源的分布式消息系统，基于高可用分布式集群技术，提供低延时的、高可靠的消息发布与订阅服务。

- Dubbo

Apache Dubbo 是一款高性能 Java RPC 框架。

- Seata

阿里巴巴开源产品，一个易于使用的高性能微服务分布式事务解决方案。

- OSS

阿里云对象存储服务（Object StorageService，简称OSS），是阿里云提供的海量、安全、低成本、高可靠的云存储服务。您可以在任何应用、任何时间、任何地点存储和访问任意类型的数据。

- SchedulerX

阿里中间件团队开发的一款分布式任务调度产品，提供秒级、精准、高可靠、高可用的定时（基于 Cron 表达式）任务调度服务。

## 4、环境和依赖

- Nacos服务

Nacos 致力于发现、配置和管理微服务。帮助开发者快速实现动态服务发现、服务配置、服务元数据及流量管理。敏捷构建、交付和管理微服务平台。首先下载 Nacos 并启动 Nacos server。

- 版本关系

版本 2.1.x.RELEASE 对应的是 SpringBoot2.1.x版本。版本2.0.x.RELEASE对应的是SpringBoot2.0.x版本。PS通常学习新的东西，最容易让人犯迷糊的地方就是版本对应关系，问题不好排查，一般都可以参考官网的给的样例。

# 二、服务中心管理

## 1、案例结构

![](https://images.gitee.com/uploads/images/2022/0208/222702_4ee7f81f_5064118.png "09-1.png")

- 服务端：`node08-nacos-server`
- 客户端：`node08-nacos-clien`

## 2、配置文件

配置内容如下：

![](https://images.gitee.com/uploads/images/2022/0208/222712_d1395eca_5064118.png "09-2.png")

```
my:
  name: cloud
  info: alibaba
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driverClassName: com.mysql.jdbc.Driver
      url: jdbc:mysql://localhost:3306/data_one
      username: root
      password: 123
```

核心内容：DataID、文件格式、文件内容的配置。这里还配置了一个MySQL数据源。

## 3、服务注册发现

**(1)、服务端配置**

- 核心依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
```

- 配置文件

```
spring:
  application:
    name: node08-nacos-server
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
```

这里的`name`配置和上面`DataID`一致，就可以读取上述配置内容。

- 提供服务接口

```java
@RestController
public class ServerWeb {
    private static Logger logger = LoggerFactory.getLogger(ServerWeb.class) ;
    @RequestMapping(value = "/web/getMsg",method = RequestMethod.GET)
    public String getMsg (@RequestParam("name") String name){
        logger.info("8001 服务被调用...");
        return "Hello：" + name ;
    }
}
```

- 启动类注解

```java
@SpringBootApplication
// SpringCloud原生注解@EnableDiscoveryClient开启服务注册发现功能
@EnableDiscoveryClient
public class ApplicationServer {
    public static void main(String[] args) {
        SpringApplication.run(ApplicationServer.class,args) ;
    }
}
```

**(2)、客户端配置**

- 核心依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
<!-- Feign组件 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>2.1.3.RELEASE</version>
</dependency>
```

这里也可以基于Feign模式，访问上述服务端提供的接口服务 。

- 基于Rest接口访问

`提供接口配置`

```
@Configuration
public class RestConfig {
    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

这里用法和SpringCloud原生框架生态相同 。

`请求方法`

```java
@RestController
public class ClientWeb {
    @Resource
    private RestTemplate restTemplate ;
    @RequestMapping(value = "/web/getMsgV1/{name}",method = RequestMethod.GET)
    public String getMsgV1 (@PathVariable String name){
        String reqUrl = "http://node08-nacos-server:8001/web/getMsg/"+name ;
        return restTemplate.getForObject(reqUrl,String.class) ;
    }
}
```

- 基于Feign接口访问

`Feign接口配置`

```java
@FeignClient("node08-nacos-server")
public interface MsgFeign {
    @GetMapping("/web/getMsg")
    String getMsg (@RequestParam(name = "name") String name);
}
```

这里用法依旧和SpringCloud原生框架生态相同 。

`请求方法`

```java
@RestController
public class ClientWeb {
    @Resource
    private MsgFeign msgFeign ;

    @RequestMapping(value = "/web/getMsgV2/{name}",method = RequestMethod.GET)
    public String getMsgV2 (@PathVariable String name){
        return msgFeign.getMsg(name) ;
    }
}
```

- 启动类配置

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages={"cloud.nacos.client.feign"})
public class ApplicationClient {
    public static void main(String[] args) {
        SpringApplication.run(ApplicationClient.class,args) ;
    }
}
```

# 三、配置中心管理

通过Nacos的配置管理可以将整合架构体系内配置，集中统一在Nacos中存储管理。分离的多环境配置，灵活的权限管理。

## 1、核心依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
```

## 2、服务端配置

```
spring:
  application:
    name: node08-nacos-server
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
```
这里的核心配置是`config`配置。

## 3、配置读取方式

这里就是Spring框架原生的读取方式。

```java
@RestController
@RefreshScope
public class ValueWeb {

    @Value("${my.name:}")
    private String myName ;

    @Value("${my.info:}")
    private String myInfo ;

    @RequestMapping("/getNameInfo")
    public String getNameInfo (){
        return myName+":"+myInfo ;
    }
}
```

## 4、数据库连接

`配置JDBC连接`

这里基于Druid连接池，配置JdbcTemplate数据库访问API。

```java
@Configuration
public class DruidConfig {

    @Value("${spring.datasource.druid.url}")
    private String dbUrl;
    @Value("${spring.datasource.druid.username}")
    private String username;
    @Value("${spring.datasource.druid.password}")
    private String password;
    @Value("${spring.datasource.druid.driverClassName}")
    private String driverClassName;

    /**
     * Druid 连接池配置
     */
    @Bean
    public DruidDataSource dataSource() {
        DruidDataSource datasource = new DruidDataSource();
        datasource.setUrl(dbUrl);
        datasource.setUsername(username);
        datasource.setPassword(password);
        datasource.setDriverClassName(driverClassName);
        return datasource;
    }
    /**
     * JDBC操作配置
     */
    @Bean(name = "jdbcTemplate")
    public JdbcTemplate jdbcTemplate (@Autowired DruidDataSource dataSource){
        return new JdbcTemplate(dataSource) ;
    }
}
```

`测试方法`

```java
@RestController
public class JdbcWeb {
    @Resource
    private JdbcTemplate jdbcTemplate ;
    @RequestMapping("/getJdbc")
    public List<String> getJdbc (){
        String sql = "select phone from d_phone" ;
        List<String> phoneEntityList = jdbcTemplate.queryForList(sql,String.class) ;
        return phoneEntityList ;
    }
}
```

# 四、案例总结

上面两个使用案例走下来，感觉和原生SpringCloud的用法区别不大，整体上手难度也不太高，毕竟AlibabaCloud框架是在SpringCloud框架基础上。适配了阿里很多开源组件，在微服务框架组件选择上，根据业务需求和团队的熟悉程序选择即可。

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node08-parent