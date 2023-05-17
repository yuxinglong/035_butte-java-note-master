
# 一、基础组件简介

## 1、Dubbo框架

Dubbo服务化治理的核心框架，之前几年在国内被广泛使用，后续由于微服务的架构的崛起，更多的公司转向微服务下成熟的技术栈，但是Dubbo本身确实是非常优秀的框架。

常见的应用迭代和升级的过程基本如下：

- 当应用访问量逐渐增大，单一应用增加机器带来的加速度越来越小，提升效率的方法之一是将应用拆成互不相干的几个应用，以提升效率。此时，用于加速前端页面开发的Web框架(MVC)是关键。
- 随着垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。此时，用于提高业务复用及整合的分布式服务框架(RPC)是关键。
- 伴随业务发展，服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。此时，用于提高机器利用率的资源调度和治理中心(SOA)是关键。

![](https://images.gitee.com/uploads/images/2022/0208/223137_320bad7e_5064118.png "11-1.png")

而Dubbo框架的核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。正好可以解决上述业务发展的痛点。

## 2、微服务框架

SpringCloud是一系列框架的有序集合。它利用SpringBoot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用SpringBoot的开发风格做到一键启动和部署。

后续AliCloud微服务系列组件也不断被使用起来，其中最基础的组件Nacos注册中心，更是直接支持Dubbo框架，这样Cloud和Dubbo两大框架就成功的整合在了一起。

## 3、Nacos注册中心

Nacos注册中心主要用于发现、配置、管理微服务。并且提供一组简单易用的特性集，快速实现动态服务发现、服务配置、服务元数据及流量管理。

![](https://images.gitee.com/uploads/images/2022/0208/223300_5001bdc7_5064118.png "11-2.png")

如上图Nacos无缝支持一些主流的开源生态框架，例如SprinCloud,Dubbo两大框架。在AliCloud的系列组件中，还包含了Seata，RocketMQ，Sentinel等一系列组件。

# 二、服务结构图解

SpringCloud和Dubbo整合的结构示意图如下，使用的Nacos中心：

![](https://images.gitee.com/uploads/images/2022/0208/223220_e8e46a12_5064118.jpeg "11-3.jpg")

**Provider提供方**：提供核心的Dubbo服务接口；

**Consumer消费方**：消费注册的Dubbo服务接口；

**Nacos注册中心**：配置、发现和管理Dubbo服务；

通过上述流程不难发现，不管从架构上看，还是用法过程，基于核心Dubbo框架和微服务原生框架是十分相似，上述流程也遵循这样一个规则：dubbo-server连接自己的业务库DB，并通过dubbo-facade中接口向外提供服务，如果不同dubbo-server需要访问其他服务接口，也必须要通过其他服务的facade接口操作，dubbo-client作为接口服务消费端，可以通过facade接口访问很多业务模块的服务，整体架构层次十分明了。

# 三、编码案例实现

## 1、案例结构和依赖

**案例结构**

![](https://images.gitee.com/uploads/images/2022/0208/223327_f43ac666_5064118.jpeg "11-4.jpg")

包含三个模块：server、facade、client。

**核心依赖**

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-dubbo</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
```

## 2、服务端配置

**配置文件**

主要是Nacos注册中心和Dubbo两个核心配置。

```
server:
  port: 9010
spring:
  application:
    name: node10-dubbo-server
  cloud:
    nacos:
      discovery:
        server-addr: http://localhost:8848
      config:
        server-addr: http://localhost:8848
        file-extension: yaml
# Dubbo服务配置
dubbo:
  scan:
    base-packages: com.cloud.dubbo.service
  protocol:
    name: dubbo
    port: -1
  registry:
    address: spring-cloud://localhost
```

**服务接口实现**

这里DubboService即dubbo-facade包中对外提供的接口。

```java
import org.apache.dubbo.config.annotation.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class DubboServiceImpl implements DubboService {

    private static final Logger LOGGER = LoggerFactory.getLogger(DubboServiceImpl.class) ;

    @Override
    public String getInfo() {
        LOGGER.info("node10-dubbo-server start ...");
        return "node10-dubbo-server";
    }
}
```

注意：@Service是Dubbo框架中的注解，不是Spring框架的注解。

## 3、消费端配置

**配置文件**

主要配置是链接Nacos注册中心，订阅注册中心的node10-dubbo-server服务。

```
server:
  port: 9011
spring:
  application:
    name: node10-dubbo-client
  cloud:
    nacos:
      discovery:
        server-addr: http://localhost:8848
      config:
        server-addr: http://localhost:8848
# Dubbo服务配置
dubbo:
  protocol:
    name: dubbo
    port: -1
  registry:
    address: spring-cloud://localhost
  cloud:
    subscribed-services: node10-dubbo-server
```

**Dubbo接口调用**

同样，这里DubboService即dubbo-facade包中对外提供的接口。

```java
import com.cloud.dubbo.service.DubboService;
import org.apache.dubbo.config.annotation.Reference;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DubboWeb {

    @Reference
    private DubboService dubboService ;

    @GetMapping("/getInfo")
    public String getInfo () {
        return dubboService.getInfo() ;
    }
}
```

注意：@Reference也是Dubbo框架中的注解。

如上流程开发完成，先后启动dubbo-server服务和dubbo-client服务，查看注册中心服务列表：

![](https://images.gitee.com/uploads/images/2022/0208/223343_cb10fb8f_5064118.jpeg "11-5.jpg")

通过上述getInfo接口请求测试，即可看到完整的案例效果。

# 四、技术选型

很少有选择SpringCloud+Dubbo框架的架构模式，这里简单说明一下为何，因为这两个框架都是相当复杂的，学习成本是一个方面，风险是最主要原因，这两个框架同时使用，就意味要面对和解决两个框架下产生的问题，在任何一个框架都可以稳定的解决业务问题时，完全没必要花里胡哨。

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node10-parent