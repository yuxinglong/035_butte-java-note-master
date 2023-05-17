# 一、熔断器简介

微服务架构特点就是多服务，多数据源，支撑系统应用。这样导致微服务之间存在依赖关系。如果其中一个服务故障，可能导致系统宕机，这就是所谓的雪崩效应。

 **1、服务熔断** 

微服务架构中某个微服务发生故障时，要快速切断服务，提示用户，后续请求，不调用该服务，直接返回，释放资源，这就是服务熔断。

熔断生效后，会在指定的时间后调用请求来测试依赖是否恢复，依赖的应用恢复后关闭熔断。

 **2、服务降级** 

服务器高并发下，压力剧增的时候,根据当业务情况以及流量，对一些服务和页面有策略的降级(可以理解为关闭不必要的服务)，以此缓解服务器资源的压力以保障核心任务的正常运行。

双十一期间，支付宝很多功能都会提示，[双十一期间，保障核心交易，某某服务数据延迟]。

 **3、核心依赖** 

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

 **4、核心注解** 

 * @EnableHystrix 启动类注解控制熔断功能。
 * @HystrixCommand 方法注解，熔断控制配置。

 **5、案例模块描述** 

```
演示基于Ribbon服务的熔断
node03-consume-8001
演示基于Feign服务的熔断
node03-consume-8002
Eureka注册中心
node03-eureka-7001
两个服务提供方
node03-provider-6001
node03-provider-6002
```

# 二、服务熔断案例

 **1、熔断执行方法** 

```
/**
 * 服务熔断调用方法
 */
public String getDefaultInfo (){
    return "服务被熔断" ;
}
```

 **2、简单案例** 

```
/**
 * 简单配置
 */
@RequestMapping("/showInfo1")
@HystrixCommand(fallbackMethod = "getDefaultInfo")
public String showInfo1 (){
    return restTemplate.getForObject(server_name+"/getInfo",String.class) ;
}
```

Hystrix默认的超时时间是1秒，超时时间内部响应，就会执行熔断，进入fallback程序。由于Spring的懒加载机制，首次请求往往比较慢，可以通过配置Hystrix超时时间解决。

 **3、复杂案例** 

配置超时、并发、线程池、指定异常熔断忽略

```
/**
 * 复杂配置
 */
@RequestMapping("/showInfo2")
@HystrixCommand(
        fallbackMethod = "getDefaultInfo",
        commandProperties={
                // 降级处理超时时间设置
                @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000"),
                // 任意时间点允许的最高并发数。超过该设置值后，拒绝执行请求。
                @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "1000"),
        },
        // 配置执行的线城池
        threadPoolProperties = {
                @HystrixProperty(name = "coreSize", value = "20"),
                @HystrixProperty(name = "maxQueueSize", value = "-1"),
        },
        // 该异常不执行熔断，去执行该异常抛出的自己逻辑
        ignoreExceptions = {ServiceException.class}
)
public String showInfo2 (){
    String value = "" ;
    // 测试配置异常不熔断
    // 响应：{"code":500,"msg":"运行异常"}
    if (value.equals("")){
        throw new ServiceException("运行异常") ;
    }
    // 该异常被熔断
    // if (value.equals("")){
    //     throw new RuntimeException("抛出错误") ;
    // }
    return restTemplate.getForObject(server_name+"/getInfo",String.class) ;
}
```

 **4、启动类注解** 

- @EnableHystrix

# 三、Feign服务熔断

 **1、Jar包说明** 

通过观察Fegin依赖的JAR可知，Fegin的Jar下包含Hystrix需要的Jar包，这里不用再次导入依赖。

 **2、熔断配置** 

Feign用接口实现的声明式Rest请求，所以配置也就在接口上面了。

- 接口代码

```
@FeignClient(value = "NODE02-PROVIDER",fallback = FallbackService.class)
public interface GetAuthorService {
    @RequestMapping(value = "/getAuthorInfo/{authorId}",method = RequestMethod.GET)
    String getAuthorInfo (@PathVariable("authorId") String authorId) ;

}
```

- 熔断执行代码

```
@Component
public class FallbackService implements GetAuthorService {
    @Override
    public String getAuthorInfo(String authorId) {
        return "服务被熔断"+authorId;
    }
}
```

- 配置文件开启熔断功能

```
feign:
  hystrix:
    enabled: true
```

 **3、服务类注解** 

由于上面的接口和熔断代码是在不同的Jar模块中，所以要在启动类@SpringBootApplication注解中扫描，如下。

```
@SpringBootApplication(scanBasePackages = {"cloud.node02.consume","cloud.block.code.service"})
@EnableEurekaClient    // 本服务启动后会自动注册进eureka服务中
@EnableDiscoveryClient
// 因为包名路径不同，需要加basePackages属性
@EnableFeignClients(basePackages={"cloud.block.code.service"})
public class Application_8002 {
    public static void main(String[] args) {
        SpringApplication.run(Application_8002.class,args) ;
    }
}
```

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node03-parent