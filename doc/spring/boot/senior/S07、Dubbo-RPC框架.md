# 一、Dubbo框架简介

## 1、框架依赖

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/143832_5235ac28_5064118.png "07-1.png")

图例说明：

- 图中小方块 Protocol, Cluster, Proxy, Service, Container, Registry, Monitor 代表层或模块，蓝色的表示与业务有交互，绿色的表示只对 Dubbo 内部交互。
- 图中背景方块 Consumer, Provider, Registry, Monitor 代表部署逻辑拓扑节点。
- 图中蓝色虚线为初始化时调用，红色虚线为运行时异步调用，红色实线为运行时同步调用。
- 图中只包含 RPC 的层，不包含 Remoting 的层，Remoting 整体都隐含在 Protocol 中。

## 2、核心角色说明

- Provider 暴露服务的服务提供方
- Consumer 调用远程服务的服务消费方（负载均衡）
- Registry 服务注册与发现的注册中心（监控、心跳、踢出、重入）
- Monitor  服务消费者和提供者在内存中累计调用次数和调用时间，主动定时每分钟发送一次统计数据到监控中心。
- Container服务运行容器：远程调用、序列化

# 二、SpringBoot2整合

## 1、核心依赖

```xml
<!-- 这里包含了Zookeeper依赖和Dubbo依赖 -->
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 2、项目结构说明

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/143919_586db434_5064118.png "07-2.png")

结构说明

```
dubbo-consume：服务消费方
dubbo-provider：服务提供方
dubbo-common：公共代码块，Dubbo接口，实体类
```

## 3、核心配置

- 提供方配置

```
server:
  tomcat:
    uri-encoding: UTF-8
    max-threads: 1000
    min-spare-threads: 30
  port: 7007
  connection-timeout: 5000ms
spring:
  application:
    name: block-dubbo-provider
# Dubbo 配置文件
dubbo:
  application:
    name: block-dubbo-provider
  registry:
    address: 127.0.0.1:2181
    protocol: zookeeper
  protocol:
    name: dubbo
    port: 20880
  scan:
    base-packages: com.boot.consume
```

- 消费方配置

```yaml
server:
  tomcat:
    uri-encoding: UTF-8
    max-threads: 1000
    min-spare-threads: 30
  port: 7008
  connection-timeout: 5000ms

spring:
  application:
    name: block-dubbo-consume
# Dubbo 配置文件
dubbo:
  application:
    name: block-dubbo-consume
  registry:
    address: 127.0.0.1:2181
    protocol: zookeeper
```

# 三、演示案例

## 1、服务远程调用

- 提供方服务接口
 * 注意这里的注解
 * com.alibaba.dubbo.config.annotation.Service
```java
@Service
@Component
public class DubboServiceImpl implements DubboService {
    private static Logger LOGGER = LoggerFactory.getLogger(DubboServiceImpl.class) ;
    @Override
    public String getInfo(String param) {
        LOGGER.info("字符参数：{}",param);
        return "[Hello,Cicada]";
    }
    @Override
    public UserEntity getUserInfo(UserEntity userEntity) {
        LOGGER.info("实体类参数：{}",userEntity);
        return userEntity;
    }
}
```

- 消费方接口

* 注意这里注解
* com.alibaba.dubbo.config.annotation.Reference
* org.springframework.stereotype.Service

```java
@Service
public class ConsumeService implements DubboService {
    @Reference
    private DubboService dubboService ;
    @Override
    public String getInfo(String param) {
        return dubboService.getInfo(param);
    }
    @Override
    public UserEntity getUserInfo(UserEntity userEntity) {
        return dubboService.getUserInfo(userEntity);
    }
}
```

## 2、接口超时配置

该配置可以在服务提供方配置，也可以在服务消费方配置，这里演示在提供方的配置。注解：timeout 

- 服务接口注解

```java
@Service(timeout = 2000)
@Component
public class DubboServiceImpl implements DubboService {
}
```

- 消费方调用

```java
 @Override
 public String timeOut(Integer time) {
     return dubboService.timeOut(time);
 }
```

- 测试接口

服务超时抛出异常

```
com.alibaba.dubbo.remoting.TimeoutException
```

## 3、接口多版本配置

- 服务提供方

相同接口提供两个版本实现。注解：version。

版本一：

```java
@Service(version = "1.0.0")
@Component
public class VersionOneImpl implements VersionService {
    @Override
    public String getVersion() {
        return "{当前版本：1.0.0}";
    }
}
```

版本二：

```java
@Service(version = "2.0.0")
@Component
public class VersionTwoImpl implements VersionService {
    @Override
    public String getVersion() {
        return "{当前版本：2.0.0}";
    }
}
```

- 消费方调用

通过@Reference(version)注解，将指向不同版本的接口实现。

```java
@Service
public class VersionServiceImpl implements VersionService {
    @Reference(version = "1.0.0")
    private VersionService versionService1 ;
    @Reference(version = "2.0.0")
    private VersionService versionService2 ;
    @Override
    public String getVersion() {
        return versionService1.getVersion();
    }
    public String version2 (){
        return versionService2.getVersion() ;
    }
}
```
以上案例都是参照Dubbo官网的流程编写的，Dubbo许多强大功能都可以参考官网一步步的配置。

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware07-dubbo-parent