
> Gateway和Netty都有盲区的感觉；

# 一、Netty简介

Netty是一个异步的，事件驱动的网络应用框架，用以快速开发高可靠、高性能的网络应用程序。

![](https://foruda.gitee.com/images/1679210014935992295/9493feb6_5064118.png "1.png")

**传输服务**：提供网络传输能力的管理；

**协议支持**：支持常见的数据传输协议；

**核心模块**：包括可扩展事件模型、通用的通信API、零拷贝字节缓冲；

# 二、Netty入门案例

## 1、服务端启动

配置Netty服务器端程序，引导相关核心组件的加载；

```java
public class NettyServer {

    public static void main(String[] args) {

        // EventLoop组，处理事件和IO
        EventLoopGroup parentGroup = new NioEventLoopGroup();
        EventLoopGroup childGroup = new NioEventLoopGroup();

        try {

            // 服务端启动引导类
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(parentGroup, childGroup)
                    .channel(NioServerSocketChannel.class).childHandler(new ChannelInit());

            // 异步IO的结果
            ChannelFuture channelFuture = serverBootstrap.bind(8082).sync();
            channelFuture.channel().closeFuture().sync();

        } catch (Exception e){
            e.printStackTrace();
        } finally {
            parentGroup.shutdownGracefully();
            childGroup.shutdownGracefully();
        }
    }
}
```

## 2、通道初始化

**ChannelInitializer**特殊的通道处理器，提供一种简单的方法，对注册到EventLoop的通道进行初始化；比如此处设置的编码解码器，自定义处理器；

```java
public class ChannelInit extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel socketChannel) {

        // 获取管道
        ChannelPipeline pipeline = socketChannel.pipeline();

        // Http编码、解码器
        pipeline.addLast("DefHttpServerCodec",new HttpServerCodec());

        // 添加自定义的handler
        pipeline.addLast("DefHttpHandler", new DefHandler());
    }
}
```

## 3、自定义处理器

处理对服务器端发起的访问，通常包括请求解析，具体的逻辑执行，请求响应等过程；

```java
public class DefHandler extends SimpleChannelInboundHandler<HttpObject> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HttpObject message) throws Exception {

        if(message instanceof HttpRequest) {
            // 请求解析
            HttpRequest httpRequest = (HttpRequest) message;
            String uri = httpRequest.uri();
            String method = httpRequest.method().name();
            log.info("【HttpRequest-URI:"+uri+"】");
            log.info("【HttpRequest-method:"+method+"】");

            Iterator<Map.Entry<String,String>> iterator = httpRequest.headers().iteratorAsString();
            while (iterator.hasNext()){
                Map.Entry<String,String> entry = iterator.next();
                log.info("【Header-Key:"+entry.getKey()+";Header-Value:"+entry.getValue()+"】");
            }

            // 响应构建
            ByteBuf content = Unpooled.copiedBuffer("Netty服务", CharsetUtil.UTF_8);
            FullHttpResponse response = new DefaultFullHttpResponse
                                        (HttpVersion.HTTP_1_1, HttpResponseStatus.OK, content);
            response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain;charset=utf-8");
            response.headers().set(HttpHeaderNames.CONTENT_LENGTH, content.readableBytes());
            ctx.writeAndFlush(response);
        }
    }
}
```

## 4、测试请求

上面入门案例中，简单的配置了一个Netty服务器端，启动之后在浏览器中模拟访问即可；

```
http://127.0.0.1:8082/?id=1&name=Spring
```

# 三、Gateway集成

## 1、依赖层级

项目中Gateway网关依赖的版本为`2.2.5.RELEASE`，发现Netty依赖的版本为`4.1.45.Final`，是当下比较主流的版本；

```xml
<!-- 1、项目工程依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>

<!-- 2、starter-gateway依赖 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-webflux</artifactId>
  <version>2.3.2.RELEASE</version>
</dependency>

<!-- 3、starter-webflux依赖 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-reactor-netty</artifactId>
  <version>2.3.2.RELEASE</version>
</dependency>
```

## 2、自动化配置

在Gateway网关的自动化配置配置类中，提供了Netty配置的管理；

```java
@AutoConfigureBefore({ HttpHandlerAutoConfiguration.class,WebFluxAutoConfiguration.class })
@ConditionalOnClass(DispatcherHandler.class)
public class GatewayAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(HttpClient.class)
    protected static class NettyConfiguration {

        @Bean
        @ConditionalOnProperty(name = "spring.cloud.gateway.httpserver.wiretap")
        public NettyWebServerFactoryCustomizer nettyServerWiretapCustomizer(
                Environment environment, ServerProperties serverProperties) {
            return new NettyWebServerFactoryCustomizer(environment, serverProperties) {
                @Override
                public void customize(NettyReactiveWebServerFactory factory) {
                    factory.addServerCustomizers(httpServer -> httpServer.wiretap(true));
                    super.customize(factory);
                }
            };
        }
    }
}
```

# 四、配置加载

## 1、基础配置

在工程的配置文件中，简单做一些基础性的设置；

```yaml
server:
  port: 8081                  # 端口号
  netty:                      # Netty组件
    connection-timeout: 3000  # 连接超时
```

## 2、属性配置类

在ServerProperties类中，并没有提供很多显式的Netty配置参数，更多信息需要参考工厂类；

```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {
    private Integer port;
    public static class Netty {
        private Duration connectionTimeout;
    }
}
```

## 3、配置加载分析

![](https://foruda.gitee.com/images/1679210024078999981/203250fc_5064118.png "2.png")

- 基于配置的属性，定制化管理Netty服务的信息；

```java
public class NettyWebServerFactoryCustomizer
        implements WebServerFactoryCustomizer<NettyReactiveWebServerFactory>{
    private final Environment environment;
    private final ServerProperties serverProperties;
    @Override
    public void customize(NettyReactiveWebServerFactory factory) {
        PropertyMapper propertyMapper = PropertyMapper.get().alwaysApplyingWhenNonNull();
        ServerProperties.Netty nettyProperties = this.serverProperties.getNetty();
        propertyMapper.from(nettyProperties::getConnectionTimeout).whenNonNull()
                .to((connectionTimeout) -> customizeConnectionTimeout(factory, connectionTimeout));
    }
}
```

- NettyReactiveWeb服务工厂，基于上述入门案例，创建WebServer时，部分参数信息出自LoopResources接口；

```java
public class NettyReactiveWebServerFactory extends AbstractReactiveWebServerFactory {

    private ReactorResourceFactory resourceFactory;

    @Override
    public WebServer getWebServer(HttpHandler httpHandler) {
        HttpServer httpServer = createHttpServer();
        ReactorHttpHandlerAdapter handlerAdapter = new ReactorHttpHandlerAdapter(httpHandler);
        NettyWebServer webServer = new NettyWebServer(httpServer, handlerAdapter, this.lifecycleTimeout);
        webServer.setRouteProviders(this.routeProviders);
        return webServer;
    }
    
    private HttpServer createHttpServer() {
		HttpServer server = HttpServer.create();
		if (this.resourceFactory != null) {
        	LoopResources resources = this.resourceFactory.getLoopResources();
        	server = server.tcpConfiguration(
        			(tcpServer) -> tcpServer.runOn(resources).addressSupplier(this::getListenAddress));
        }
        return applyCustomizers(server);
	}
	
}
```

# 五、周期管理方法

## 1、控制类

![](https://foruda.gitee.com/images/1679210032988135446/08c9f598_5064118.png "3.png")

Gateway项目中，Netty服务核心控制类，通过NettyReactiveWebServerFactory工厂类创建，对Netty生命周期的管理提供了一层包装；

```java
public class NettyWebServer implements WebServer {

    private final HttpServer httpServer;
    private final ReactorHttpHandlerAdapter handlerAdapter;

    /**
     * 启动方法
     */
    @Override
    public void start() throws WebServerException {
        if (this.disposableServer == null) {
            this.disposableServer = startHttpServer();
            // 控制台日志
            logger.info("Netty started on port(s): " + getPort());
            startDaemonAwaitThread(this.disposableServer);
        }
    }
    private DisposableServer startHttpServer() {
        HttpServer server = this.httpServer;
        if (this.routeProviders.isEmpty()) {
            server = server.handle(this.handlerAdapter);
        }
        return server.bindNow();
    }

    /**
     * 停止方法
     */
    @Override
    public void stop() throws WebServerException {
        if (this.disposableServer != null) {
            // 释放资源
            if (this.lifecycleTimeout != null) {
                this.disposableServer.disposeNow(this.lifecycleTimeout);
            }
            else {
                this.disposableServer.disposeNow();
            }
            // 对象销毁
            this.disposableServer = null;
        }
    }
}
```

## 2、管理类

Netty组件中抽象管理类，以安全的方式构建Http服务；

```java
public abstract class HttpServer {

    public static HttpServer create() {
        return HttpServerBind.INSTANCE;
    }

    public final DisposableServer bindNow() {
        return bindNow(Duration.ofSeconds(45));
    }

    public final HttpServer handle(BiFunction<? super HttpServerRequest, ? super
            HttpServerResponse, ? extends Publisher<Void>> handler) {
        return new HttpServerHandle(this, handler);
    }
}
```

# 六、参考源码

```
编程文档：
https://gitee.com/cicadasmile/butte-java-note

应用仓库：
https://gitee.com/cicadasmile/butte-flyer-parent
```