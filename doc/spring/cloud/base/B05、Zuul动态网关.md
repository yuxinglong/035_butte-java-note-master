# 一、Zuul组件简介

**1、基础概念** 

Zuul 网关主要提供动态路由，监控，弹性，安全管控等功能。在分布式的微服务系统中，系统被拆为了多个微服务模块，通过zuul网关对用户的请求进行路由，转发到具体的后微服务模块中。

**2、Zuul的作用** 

- 按照不同策略，将请求转发到不同的服务上去；
- 聚合API接口，统一对外暴露，提高系统的安全性；
- 实现请求统一的过滤，以及服务的熔断降级；

**3、案例结构** 

![](https://images.gitee.com/uploads/images/2022/0207/231844_66e5cec6_5064118.jpeg "05-1.jpg")

启动顺序如下：
```
# 注册中心
node05-eureka-7001
# 两个服务提供者
node05-provider-6001
node05-provider-6002
# 网关控制
node05-zuul-7002
```

启动成功后，注册中心展示如下：

![](https://images.gitee.com/uploads/images/2022/0207/231855_bf61403c_5064118.jpeg "05-2.jpg")

# 二、Zuul使用详解

**1、核心依赖** 

```
<!-- 路由网关 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

**2、核心配置文件** 

```
server:
  port: 7002
spring:
  application:
    name: cloud-node05-parent
eureka:
  instance:
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://registry01.com:7001/eureka/
zuul:
  # 前缀，可以用来做版本控制
  prefix: /v1
  # 禁用默认路由，执行配置的路由
  ignored-services: "*"
  routes:
    # 配置6001接口微服务
    pro6001:
      serviceId: node05-provider-6001
      path: /api-6001/**
    # 配置6002接口微服务
    pro6002:
      serviceId: node05-provider-6002
      path: /api-6002/**
```

- 启动类注解：@EnableZuulProxy

**3、统一服务降级** 

实现FallbackProvider接口，自定义响应提示。

```
@Component
public class FallBackConfig implements FallbackProvider {
    private static final Logger LOGGER = LoggerFactory.getLogger(FallBackConfig.class) ;
    @Override
    public ClientHttpResponse fallbackResponse(Throwable cause) {
        // 捕获超时异常，返回自定义信息
        if (cause instanceof HystrixTimeoutException) {
            return response(HttpStatus.GATEWAY_TIMEOUT);
        } else {
            return fallbackResponse();
        }
    }
    private ClientHttpResponse response(final HttpStatus status) {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() {
                return status;
            }
            @Override
            public int getRawStatusCode() {
                return status.value();
            }
            @Override
            public String getStatusText() {
                return status.getReasonPhrase();
            }
            @Override
            public void close() {
                LOGGER.info("close");
            }
            @Override
            public InputStream getBody() {
                String message =
                        "{\n" +
                            "\"code\": 200,\n" +
                            "\"message\": \"微服务飞出了地球\"\n" +
                        "}";
                return new ByteArrayInputStream(message.getBytes());
            }
            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
    @Override
    public String getRoute() {
        return "*";
    }
    @Override
    public ClientHttpResponse fallbackResponse() {
        return response(HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

**4、统一过滤器** 

继承ZuulFilter类，自定义过滤动作。

```
@Component
public class FilterConfig extends ZuulFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(FilterConfig.class) ;
    @Override
    public String filterType() {
        return "pre";
    }
    @Override
    public int filterOrder() {
        return 0;
    }
    @Override
    public boolean shouldFilter() {
        return true;
    }
    @Override
    public Object run() {
        RequestContext requestContext = RequestContext.getCurrentContext() ;
        try {
            doBizProcess(requestContext);
        } catch (Exception e){
            LOGGER.info("异常：{}",e.getMessage());
        }
        return null;
    }

    public void doBizProcess (RequestContext requestContext) throws Exception {
        HttpServletRequest request = requestContext.getRequest() ;
        String reqUri = request.getRequestURI() ;
        if (!reqUri.contains("getAuthorInfo")){
            requestContext.setSendZuulResponse(false);
            requestContext.setResponseStatusCode(401);
            requestContext.getResponse().getWriter().print("Path Is Error...");
        }
    }
}
```

**5、测试流程** 

- 测试网关配置

访问如下接口，响应正常，说明网关配置生效：
```
http://localhost:7002/v1/api-6001/getAuthorInfo/1
http://localhost:7002/v1/api-6002/getAuthorInfo/2
```

- 测试服务降级

关闭6001服务，再次访问接口，提示信息如下，说明服务降级策略生效：
```
{
	"code": 200,
	"message": "微服务飞出了地球"
}
```

- 测试过滤器

因为请求URI不匹配getAuthorInfo，所以被拦截，说明过滤器略生效：

```
http://localhost:7002/v1/api-6001/
响应提示：
Path Is Error...
```

**源码参考：** https://gitee.com/cicadasmile/spring-cloud-base/tree/master/cloud-node05-parent