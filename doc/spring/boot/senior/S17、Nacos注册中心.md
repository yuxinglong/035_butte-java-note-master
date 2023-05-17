# 一、Nacos基础简介

## 1、概念简介

Nacos 是构建以“服务”为中心的现代应用架构，如微服务范式、云原生范式等服务基础设施。聚焦于发现、配置和管理微服务。Nacos提供一组简单易用的特性集，帮助开发者快速实现动态服务发现、服务配置、服务元数据及流量管理。敏捷构建、交付和管理微服务平台。 

## 2、关键特性

- 动态配置服务
- 服务发现和服务健康监测
- 动态 DNS 服务
- 服务及其元数据管理

## 3、专业术语解释

- 命名空间

用于进行租户粒度的配置隔离。不同的命名空间下，可以存在相同的 Group 或 Data ID 的配置。

- 配置集

一组相关或者不相关的配置项的集合称为配置集。在系统中，一个配置文件通常就是一个配置集，包含了系统各个方面的配置。

- 配置集 ID

Nacos 中的某个配置集的ID。配置集ID是组织划分配置的维度之一。DataID通常用于组织划分系统的配置集。

- 配置分组

Nacos 中的一组配置集，是组织配置的维度之一。通过一个有意义的字符串对配置集进行（Group）分组，从而区分 Data ID 相同的配置集。

- 配置快照

Nacos 的客户端 SDK 会在本地生成配置的快照。当客户端无法连接到 Nacos Server 时，可以使用配置快照显示系统的整体容灾能力。

- 服务注册

存储服务实例和服务负载均衡策略的数据库。

- 服务发现

使用服务名对服务下的实例的地址和元数据进行探测，并以预先定义的接口提供给客户端进行查询。

- 元数据

Nacos数据（如配置和服务）描述信息，如服务版本、权重、容灾策略、负载均衡策略等。

## 4、Nacos生态圈

Nacos 无缝支持一些主流的开源框架生态：

- Spring Cloud 微服务框架 ;
- Dubbo RPC框架 ;
- Kubernetes 容器应用 ;

# 二、Nacos环境搭建

## 1、环境版本

这里在Windos环境下搭建Nacos单个服务。

- Nacos版本：官方推荐的稳定版本为1.1.4。
- 基础环境：JDK 1.8+；Maven 3.2.x

## 2、环境包下载

这里直接下载打包好的文件，也可以下载源码自己打包。

> https://github.com/alibaba/nacos/releases

下载文件：nacos-server-1.1.4.zip

## 3、启动环境

- 启动文件地址：`nacos\bin`
- 启动文件：`startup.cmd`
- 关闭文件：`shutdown.cmd`

启动后登陆，账户密码默认：nacos/nacos ;首页效果如下：

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/151811_a8f7c784_5064118.png "17-1.png")

# 三、整合SpringBoot2

注意：版本 0.2.x.RELEASE 对应的是 Spring Boot 2.x 版本，版本 0.1.x.RELEASE 对应的是 Spring Boot 1.x 版本。

## 1、新建配置

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/151821_9488374f_5064118.png "17-2.png")

## 2、核心依赖

```xml
<!-- Nacos 组件依赖 -->
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-discovery-spring-boot-starter</artifactId>
    <version>0.2.3</version>
</dependency>
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-config-spring-boot-starter</artifactId>
    <version>0.2.3</version>
</dependency>
```

## 3、Yml配置文件

这里把项目作为服务注册到Nacos中。

```yaml
nacos:
  config:
    server-addr: 127.0.0.1:8848
  discovery:
    server-addr: 127.0.0.1:8848
```

## 4、启动类配置

启动类关联配置中心的dataId标识。

```java
@EnableSwagger2
@SpringBootApplication
@NacosPropertySource(dataId = "WARE_ID", autoRefreshed = true)
public class Application7017 {
    public static void main(String[] args) {
        SpringApplication.run(Application7017.class,args) ;
    }
}
```

## 5、核心配置类

```java
import com.alibaba.nacos.api.annotation.NacosInjected;
import com.alibaba.nacos.api.exception.NacosException;
import com.alibaba.nacos.api.naming.NamingService;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import javax.annotation.PostConstruct;
@Configuration
public class NacosConfig {
    @Value("${server.port}")
    private int serverPort;
    @Value("${spring.application.name}")
    private String applicationName;
    @NacosInjected
    private NamingService namingService;
    @PostConstruct
    public void registerInstance() throws NacosException {
        namingService.registerInstance(applicationName, "127.0.0.1", serverPort);
    }
}
```

启动成功后查询服务列表：

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/151829_bc1ac1f0_5064118.png "17-3.png")

## 6、基础API用例

这里演示两个基础用法：上述步骤1的配置内容读取，步骤4的服务列表读取。基于swagger2管理测试接口。

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/151840_c76a101a_5064118.png "17-4.png")

```java
@Api("Nacos接口管理")
@RestController
@RequestMapping("/nacos")
public class NacosController {

    @NacosValue(value = "${MyName:null}", autoRefreshed = true)
    private String myName;
    @NacosValue(value = "${project:null}", autoRefreshed = true)
    private String project;

    @ApiOperation(value="查询配置信息")
    @GetMapping(value = "/info")
    public String info () {
        return myName+":"+project;
    }

    @NacosInjected
    private NamingService namingService;

    @ApiOperation(value="查询服务列表")
    @GetMapping(value = "/getServerList")
    public List<Instance> getServerList (@RequestParam String serviceName) {
        try {
            return namingService.getAllInstances(serviceName) ;
        } catch (Exception e){
            e.printStackTrace();
        }
        return null ;
    }
}
```

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware17-regist-nacos