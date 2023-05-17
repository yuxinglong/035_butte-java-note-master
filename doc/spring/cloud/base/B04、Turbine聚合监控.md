# 一、聚合监控简介

 **1、Dashboard组件** 

微服务架构中为了保证程序的可用性，防止程序出错导致网络阻塞，出现了断路器模型。断路器的状况反应程序的可用性和健壮性，它是一个重要指标。HystrixDashboard是作为断路器状态的一个组件，提供了数据监控和直观的图形化界面。

 **2、Turbine组件** 

Hystrix Dashboard组件监控服务的熔断情况时,每个服务都有图形界面,当微服务数量很多时,监控非常繁杂.为了同时监控多个服务的熔断状况,Netflix开源了Hystrix的另一个组件Turbine.Turbine用于聚合多个Hystrix Dashboard监控,将多个Hystrix Dashboard组件的数据聚集在一个面板展示,集中监控。 

 **3、案例结构** 

![](https://images.gitee.com/uploads/images/2022/0207/230347_786f2c83_5064118.png "04-1.png")

```
聚合监控服务
node04-monitor-7002
注册中心
node04-eureka-7001
两个服务提供者，都配置了熔断器，和Dashboard组件
node04-provider-6001
node04-provider-6002
```

# 二、Dashboard组件

这个组件是针对单个微服务的监控的。具体使用流程如下。

 **1、注解和依赖** 

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```

启动类注解
* @EnableHystrix
* @EnableHystrixDashboard

 **2、启动下面两个服务** 

```
node04-eureka-7001
node04-provider-6001
```

 **3、访问指定接口** 

- 访问配置的熔断接口

`http://localhost:6001/getInfo`

- 打开数据面板

`http://localhost:6001/hystrix.stream`

可以看到一些具体的数据，类似打印日志的方式，展现上面接口的执行信息。

- 打开图形面板

http://localhost:6001/hystrix

查看配置监控信息。

![](https://images.gitee.com/uploads/images/2022/0207/230407_0559e302_5064118.png "04-2.png")

刷新几次上面配置的熔断接口，查看效果。

![](https://images.gitee.com/uploads/images/2022/0207/230419_8a087ec6_5064118.png "04-3.png")

# 三、Turbine组件

> node04-monitor-7002 聚合监控服务，聚集6001，和6002两个服务的监控。

 **1、依赖和注解** 

- 服务提供者新增依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

- 聚合服务依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-turbine</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-netflix-turbine</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

启动类注解

* @EnableTurbine

 **2、启动服务** 

依次启动注册中心，两个服务提供者，最后启动聚合监控中心。

 **3、操作流程** 

- 打开监控面板

进行如下配置

![](https://images.gitee.com/uploads/images/2022/0207/230433_94487b47_5064118.png "04-4.png")

- 刷新两个服务的熔断接口

```
http://localhost:6001/getInfo
http://localhost:6002/getInfo
```

- 查看上面面板的监控信息如下。

![](https://images.gitee.com/uploads/images/2022/0207/230448_ae43713a_5064118.png "04-5.png")

聚合监控服务流程就是这样了。

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node04-parent