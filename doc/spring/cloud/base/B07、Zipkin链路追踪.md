# 一、链路追踪简介

**1、Sleuth组件简介** 

Sleuth是SpringCloud微服务系统中的一个组件，实现了链路追踪解决方案。可以定位一个请求到底请求了哪些具体的服务。在复杂的微服务系统中，如果请求发生了异常，可以快速捕获问题所在的服务。

**2、项目结构** 

启动顺序如下
```
* 注册中心
node07-eureka-7001
* 链路数据收集服务
node07-zipkin-7003
* 服务提供
node07-provider-6001
node07-provider-6002
* 网关路由
node07-zuul-7002
```

# 二、搭建链路服务

**1、核心依赖** 

```
<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-server</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-autoconfigure-ui</artifactId>
</dependency>
```

- 启动类注解：@EnableZipkinServer

 **2、配置文件** 

```
server:
  port: 7003
spring:
  application:
    name: node07-zipkin-7003
eureka:
  instance:
    hostname: zipkin-7003
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://registry01.com:7001/eureka/
```

# 三、服务配置

这里网关，zuul-7002，服务提供，provider-6001，provider-6002的配置相同。

 **1、核心依赖** 

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

 **2、配置文件** 

```
spring:
  zipkin:
    base-url: http://localhost:7003
  sleuth:
    sampler:
      # 数据 100% 上传
      percentage: 1.0
```

# 四、测试流程

 **1、注册中心** 

一次启动上述服务之后，查看注册中心：

![](https://images.gitee.com/uploads/images/2022/0207/232825_cf2c5539_5064118.jpeg "07-1.jpg")

 **2、请求流程** 

访问接口

```
http://localhost:7002/v1/api-6001/get6001Info
```

这个请求从网关服务进入，到达6001端口服务之后，请求6002端，最终返回结果。

- 6001接口

```
@Autowired
private RestTemplate restTemplate ;
@RequestMapping("/get6001Info")
public String get6001Info (){
    String server_name = "http://node07-provider-6002" ;
    return restTemplate.getForObject(server_name+"/get6002Info",String.class) ;
}
```
- 6002接口
```
@RequestMapping(value = "/get6002Info",method = RequestMethod.GET)
public String get6002Info () {
    LOG.info("provider-6002");
    return "6002Info" ;
}
```

 **3、链路管理界面** 

- UI界面

访问接口

```
http://localhost:7003/zipkin/
```

![](https://images.gitee.com/uploads/images/2022/0207/232841_fd3565cd_5064118.jpeg "07-2.jpg")

- 依赖分析

如图点击，【依赖分析】，和上面描述的请求过程完全一致。

![](https://images.gitee.com/uploads/images/2022/0207/232853_6d020f53_5064118.jpeg "07-3.jpg")

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node07-parent