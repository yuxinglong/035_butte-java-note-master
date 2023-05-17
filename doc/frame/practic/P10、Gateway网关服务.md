> 有多少请求，被网关截胡；

# 一、Gateway简介

微服务架构中，网关服务通常提供动态路由，以及流量控制与请求识别等核心能力，在之前的篇幅中有说过Zuul组件的使用流程，但是当下Gateway组件是更常规的选择，下面就围绕Gateway的实践做详细分析；

![](https://foruda.gitee.com/images/1662989182214862118/56ef6631_5064118.png "01.png")

从架构模式上看，网关不管采用什么技术组件，都是在客户端与业务服务中间提供一层拦截与校验的能力，但是相比较Zuul来说，Gateway提供了更强大的功能和卓越的性能；

基于实践的场景来看，在功能上网关更侧重请求方的合法校验，流量管控，以及IP级别的拦截，从架构层面看，通常需要提供灵活的路由机制，比如灰度，负载均衡的策略等，并基于消息机制，进行系统级的安全通知等；

![](https://foruda.gitee.com/images/1662989195281630803/2c7fe976_5064118.png "02.png")

下面围绕客户端、网关层、门面服务的三个节点，分析Gateway的使用细节，即客户端向网关发出请求，经过网关路由到门面服务处理；

# 二、动态路由

## 1、基础概念

**路由**：作为网关中最核心的能力，从源码结构上看，包括ID、请求URI、断言集合、过滤集合等组成；

```java
public class RouteDefinition {
	private String id;
	private URI uri;
	private List<PredicateDefinition> predicates = new ArrayList<>();
	private List<FilterDefinition> filters = new ArrayList<>();
}
```

**断言+过滤**：通常在断言中定义请求的匹配规则，在过滤中定义请求的处理动作，结构上看都是名称加参数集合，并且支持快捷的方式配置；

```java
public class PredicateDefinition {
	private String name;
	private Map<String, String> args = new LinkedHashMap<>();
}

public class FilterDefinition {
	private String name;
	private Map<String, String> args = new LinkedHashMap<>();
}
```

## 2、配置路由

以配置的方式，添加`facade`服务路由，以路径匹配的方式，如果请求路径错误则断言失败，StripPrefix设置为1，即在过滤中去掉第一个`/facade`参数；

```YAML
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: facade
          uri: http://127.0.0.1:8082
          predicates:
            - Path=/facade/**
          filters:
            - StripPrefix=1
```

执行原理如下：

![](https://foruda.gitee.com/images/1662989203861682886/3f6ab0d1_5064118.png "03.png")

这里是以配置文件的方式，设置`facade`服务的路由策略，其中指定了路径方式，在Gateway文档中提供了多种路由样例，比如：Header、Cookie、Method、Query、Host等断言方式；

## 3、编码方式

基于编码的方式管理路由策略，在Gateway文档同样提供了多种参考样例，如果路由服务少并且固定，配置的方式可以解决，如果路由服务很多，并且需要动态添加，那基于库表方式更适合；

```java
@Configuration
public class GateConfig {
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("facade",r -> r.path("/facade/**").filters(f -> f.stripPrefix(1))
                .uri("http://127.0.0.1:8082")).build();
    }
}
```

## 4、库表加载

在常规的应用中，从库表中读取路由策略是比较常见的方式，定义路由工厂类并实现`RouteDefinitionRepository`接口，涉及加载、添加、删除三个核心方法，然后基于服务类从库中读取数据转换为`RouteDefinition`对象即可；

![](https://foruda.gitee.com/images/1662989212669265798/c4186d6d_5064118.png "04.png")

```java
@Component
public class DefRouteFactory implements RouteDefinitionRepository {
    @Resource
    private ConfigRouteService routeService ;
    // 加载
    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routeService.getRouteDefinitions());
    }
    // 添加
    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(routeDefinition -> { routeService.saveRouter(routeDefinition);
            return Mono.empty();
        });
    }
    // 删除
    @Override
    public Mono<Void> delete(Mono<String> idMono) {
        return idMono.flatMap(routeId -> { routeService.removeRouter(routeId);
            return Mono.empty();
        });
    }
}
```

在源码仓库中采用的就是库表管理的方式，代码逻辑的更多细节可以移步Git参考，此处不再过多粘贴；

# 三、自定义路由策略

- **自定义断言**，继承`AbstractRoutePredicateFactory`类，注意命名以`RoutePredicateFactory`结尾，重写`apply`方法，即可执行特定的匹配规则；

```java
@Component
public class DefCheckRoutePredicateFactory extends AbstractRoutePredicateFactory<DefCheckRoutePredicateFactory.Config> {
    public DefCheckRoutePredicateFactory() {
        super(Config.class);
    }
    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return new GatewayPredicate() {
            @Override
            public boolean test(ServerWebExchange serverWebExchange) {
                log.info("DefCheckRoutePredicateFactory：" + config.getName());
                return StrUtil.equals("butte",config.getName());
            }
        };
    }
    @Data
    public static class Config { private String name; }
    @Override
    public List<String> shortcutFieldOrder() { return Collections.singletonList("name"); }
}
```

- **自定义过滤**，继承`AbstractNameValueGatewayFilterFactory`类，注意命名以`GatewayFilterFactory`结尾，重写`apply`方法，即可执行特定的过滤规则；

```java
@Component
public class DefHeaderGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {
    @Override
    public GatewayFilter apply(AbstractNameValueGatewayFilterFactory.NameValueConfig config) {
        return (exchange, chain) -> {
            log.info("DefHeaderGatewayFilterFactory："+ config.getName() + "-" + config.getValue());
            return chain.filter(exchange);
        };
    }
}
```

- **配置加载方式**，此处断言与过滤即快捷的配置方式，所以在命名上要遵守Gateway的约定；

```
spring:
  cloud:
    gateway:
      routes:
        - id: facade
          uri: http://127.0.0.1:8082
          predicates:
            - Path=/facade/**
            - DefCheck=butte
          filters:
            - StripPrefix=1
            - DefHeader=cicada,smile
```

通常来说，在应用级的系统中都需要进行断言和过滤的策略自定义，以提供业务或者架构层面的支撑，完成更加细致的规则校验，尤其在相同服务多版本并行时，可以更好的管理路由策略，从而避免分支之间的影响；

# 四、全局过滤器

在路由中采用的过滤是`GatewayFilter`，实际Gateway中还提供了`GlobalFilter`全局过滤器，虽然从结构上看十分相似，但是其职责是有本质区别的；

- 全局过滤器1：打印请求ID

```java
@Component
@Order(1)
public class DefOneGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("request-id:{}",exchange.getRequest().getId()) ;
        return chain.filter(exchange);
    }
}
```

- 全局过滤器2：打印请求URI

```java
@Component
@Order(2)
public class DefTwoGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("request-uri:{}",exchange.getRequest().getURI()) ;
        return chain.filter(exchange);
    }
}
```

Gateway网关作为微服务架构系统中最先接收请求的一层，可以定义许多策略来保护系统的安全，比如高并发接口的限流，第三方授权验证，遭到恶意攻击时的IP拦截等等，尽量将非法请求在网关中拦截掉，从而保证系统的安全与稳定。

# 五、参考源码

```
应用仓库：
https://gitee.com/cicadasmile/butte-flyer-parent

组件封装：
https://gitee.com/cicadasmile/butte-frame-parent
```