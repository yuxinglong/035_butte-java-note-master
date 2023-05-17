
# 一、基本简介

## 1、概念描述

Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。包括核心的独立类库，监控台，丰富的使用场景验证。（这似乎是阿里开源组件的一贯作风，极其有特点，且特点很规律）

基本特性图：

![](https://images.gitee.com/uploads/images/2022/0208/222912_2232343b_5064118.png "10-1.png")

`补刀一句`：这种图很多人可能不在意，但是一般官方给这个图就是该中间件的基本使用思路，与核心功能点。

## 2、基础性概念

- 资源管理

资源是Sentinel组件中的核心概念之一。应用服务器上脚本，静态页面，API接口，文件图片等都可以理解为资源，对于Java开发者而言，API接口就是这里资源的概念。 

- 规则配置

Sentinel组件通过流控规则的配置，来指定允许该资源（API接口）通过的请求次数，IP黑白名单，应用服务等。

- 测试效果

`QPS`：每秒查询率，是一台服务器每秒能够处理的查询次数。

`TPS`：每秒处理事务数，事务处理整体倾向于整个过程。

# 二、框架环境整合

这里的环境主要整合Nacos注册中心，Feign服务，Sentinel哨兵，和Sentinel控制台。

## 1、基本依赖

这里的依赖需要参考官方文档，不同的环境使用不同的依赖，这里主要适配SpringCloud环境，所以使用如下包即可。

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
```

## 2、控制台面板

这里直接从GitHub下载一个控制台服务包即可，也可以自己下载源码，按照需求修改后自行打包。

```
java -jar sentinel-dashboard-1.7.1.jar
```

下载并启动控制台服务。

## 3、服务配置

这里主要是把用到的两个服务9001和9002连接到监控台。

```
spring:
  application:
    name: node09-nacos-9001
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
    sentinel:
      transport:
        port: 9001
        dashboard: localhost:8080
```

最下面四行文件是哨兵控制台的主要配置，注意刚启动之后控制台是看不到连接的，有资源被触发之后才能看到。(附一张首页效果图)

![](https://images.gitee.com/uploads/images/2022/0208/222927_9695f227_5064118.jpeg "10-2.jpg")

# 三、流量控制

## 1、基本描述

流量控制（flow control），其原理是监控应用流量的 QPS 或并发线程数等指标，当达到指定的阈值时对流量进行控制，以避免被瞬时的流量高峰冲垮，从而保障应用的高可用性。

## 2、限流规则

限流规则主要由下面几个因素组成。

- resource：资源名，即限流规则的作用对象，对于Java服务端开发而言就是执行的方法；
- count: 限流阈值，单位时间内能按照规则通过的请求量；
- grade: 限流阈值类型，QPS 或并发线程数 ；
- limitApp: 流控限制的指定应用来源，若为default则不区分调用来源；
- strategy: 调用关系限流策略，直连，链路等；
- controlBehavior: 流量控制效果，直接拒绝、Warm Up、匀速排队；

## 3、基本案例

- 硬编码

`配置规则`

```java
public class FlowRuleConfig {

    public static void initFlowQpsRule(String resourceName) {
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule(resourceName);
        // 修改这里参数，查看效果
        rule.setCount(100);
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setLimitApp("default");
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
    }
}
```

该规则参考上面的限流因素，不难理解。

`使用方式`

```java
@RequestMapping(value = "/web/getOrder",method = RequestMethod.GET)
public String getOrder (@RequestParam("id") Integer id){
    FlowRuleConfig.initFlowQpsRule("getOrder");
    Entry entry = null;
    try {
        // 定义资源，埋点
        entry = SphU.entry("getOrder");
        // 保护的业务逻辑
        return "Order=" + id ;
    } catch (Exception e){
        e.printStackTrace();
    } finally {
        if (entry != null){entry.exit();}
    }
    return "Order Error" ;
}
```

测试的时候修改规则中count值，测试效果明显。

- 编码简化

SphU.entry中可以设置处理类型，限流阈值。

```java
@RequestMapping(value = "/web/getState",method = RequestMethod.GET)
public String getState (@RequestParam("id") Integer id){
    Entry entry = null;
    try {
        entry = SphU.entry("getOrder",EntryType.IN,2);
        return "state=" + id;
    }
    catch (BlockException e){
        e.printStackTrace();
    } finally {
        if (entry != null){entry.exit();}
    }
    return "State Error" ;
}
```

不过这种模式依旧是代码入侵严重，不太符合现在编程的大趋势。Sentinel支持通过@SentinelResource注解定义资源并配置。

## 4、测试效果

请求上述的两个测试接口，之后看控制台中9001服务的簇点链路。

![](https://images.gitee.com/uploads/images/2022/0208/222943_2186f28d_5064118.jpeg "10-3.jpg")

可以基于控制台实时配置资源的：流控、降级、热点、授权等核心功能，服务重启之后配置也会重置。

# 四、服务熔断降级

## 1、注解详解

**核心注解`SentinelResource`**

用于定义资源，并提供可选的异常处理和 fallback 配置项。 @SentinelResource 注解包含以下属性：

- value

资源名称，核心概念不能为空;

- entryType

entry 类型，可选项默认为 EntryType.OUT;

- blockHandler

对应处理BlockException的函数名称，可选项。blockHandler函数访问范围需要是public，返回类型需要与原方法相匹配，参数类型需要和原方法相匹配并且最后加一个额外的参数，类型为BlockException。blockHandler函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 blockHandlerClass为对应的类的Class对象，注意对应的函数必须为 static 函数，否则无法解析。

- fallback

fallback函数名称，可选项，用于在抛出异常的时候提供fallback处理逻辑。fallback函数可以针对所有类型的异常（除了 exceptionsToIgnore里面排除掉的异常类型）进行处理。fallback 函数签名和位置要求：返回值类型必须与原函数返回值类型一致；方法参数列表需要和原函数一致，或者可以额外多一个;

`注意`：这里可以这样理解blockHandler和fallback，fallback处理业务逻辑的异常，服务降级，blockHandler处理sentinel组件控制的阻断异常。

## 2、使用案例

**9001接口服务**

```java
@Service
public class FlowServiceImpl implements FlowService {
    @Resource
    private JdbcTemplate jdbcTemplate ;
    @SentinelResource(value = "getPhone",blockHandler = "exceptionHandler", fallback = "fallback")
    @Override
    public String getPhone(Integer id) {
        String sql = "select phone from d_phone WHERE id="+id ;
        Integer.parseInt("hand") ;
        return jdbcTemplate.queryForList(sql,String.class).get(0) ;
    }
    // 降级处理
    public String fallback(Integer id) {
        return "服务降级,id="+id ;
    }
    // 异常处理
    public String exceptionHandler(Integer id,BlockException be) {
        be.printStackTrace();
        return "服务抛异常,id="+id ;
    }
}
```

**9002请求服务**

```java
@RequestMapping(value = "/web/getJdbc",method = RequestMethod.GET)
public String getJdbc () {
    return msgFeign.getJdbc() ;
}
```

## 3、效果测试

通过控制台配置9001的限流规则，然后刷接口看效果。

![](https://images.gitee.com/uploads/images/2022/0208/222958_b03aef36_5064118.jpeg "10-4.jpg")

注意阻断异常和业务异常的返参区别。

## 4、对比Hystrix组件

Hystrix的核心点在于以隔离和熔断为主的容错机制，超时或被熔断的调用将会快速失败，并可以提供 fallback 机制；

Sentinel核心点在于流量控制多样化，熔断降级服务，系统负载保护，实时监控和控制台；

`补刀一句`：对于流量控制类的组件，大部分场景是使用限流，服务降级这两块核心功能。

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node09-parent