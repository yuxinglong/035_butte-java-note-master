# 一、Ehcache缓存简介

## 1、基础简介

EhCache是一个纯Java的进程内缓存框架，具有快速、上手简单等特点，是Hibernate中默认的缓存提供方。

## 2、Hibernate缓存

Hibernate三级缓存机制简介：

**一级缓存**：基于Session级别分配一块缓存空间，缓存访问的对象信息。Session关闭后会自动清除缓存。

**二级缓存**：是SessionFactory对象缓存，可以被创建出的多个 Session 对象共享，二级缓存默认是关闭的，如果要使用需要手动开启，并且依赖EhCache组件。

**三级缓存**：查询缓存，配置开启该缓存的情况下，重复使用一个sql查询某个范围内的数据，会进行缓存。

## 3、EhCache缓存特点

- 快速，简单，并且提供多种缓存策略；
- 缓存数据有两级：内存和磁盘，无需担心容量问题；
- 缓存数据会在虚拟机重启的过程中写入磁盘；
- 可以通过RMI、可插入API等方式进行分布式缓存；
- 具有缓存和缓存管理器的侦听接口；
- 支持多缓存管理器实例，以及一个实例的多个缓存区域；
- 提供Hibernate的缓存实现；

## 4、对比Redis缓存

**Ehcache**：直接在Jvm虚拟机中缓存，速度快，效率高，不适合处理大规模缓存数据，在分布式环境下，缓存数据共享操作复杂；

**Redis**：作为独立的缓存中间件，在分布式缓存系统中非常好用，缓存数据共享，有效支撑大量数据缓存，支持哨兵模式，或者集群模式的高可用成熟方案；

# 二、集成SpringBoot框架

## 1、核心依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

## 2、加载配置

**基础配置**

```yaml
spring:
  cache:
    ehcache:
      config: classpath:ehcache.xml
```

**启动类注解**

```java
@EnableCaching
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class,args) ;
    }
}
```

## 3、配置详解

```xml
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="../config/ehcache.xsd">

    <!-- 操作系统缓存的临时目录,内存满后写入该目录 -->
    <diskStore path="java.io.tmpdir"/>

    <defaultCache
            maxElementsInMemory="1000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            maxElementsOnDisk="10000000"
            diskExpiryThreadIntervalSeconds="120"
            memoryStoreEvictionPolicy="LRU">
        <persistence strategy="localTempSwap"/>
    </defaultCache>

    <cache name="userEntity"
           maxElementsInMemory="1000"
           eternal="false"
           timeToIdleSeconds="120"
           timeToLiveSeconds="120"
           maxElementsOnDisk="10000000"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU">
        <persistence strategy="localTempSwap"/>
    </cache>
</ehcache>
```

**配置参数说明**

**maxElementsOnDisk**:磁盘缓存中最多可以存放的元素数量;

**eternal**:缓存中对象是否永久有效;

**timeToIdleSeconds**:当eternal=false时使用，缓存数据有效期(单位:秒),时间段内没有访问该元素,将被清除;

**timeToLiveSeconds**:缓存数据的存活时间;

**maxElementsInMemory**:内存中最多可以存放的元素数量,overflowToDisk=true,则会将Cache中多出的元素放入磁盘文件中,若overflowToDisk=false,则根据memoryStoreEvictionPolicy策略替换Cache中原有的元素;

**diskExpiryThreadIntervalSeconds**:磁盘缓存的清理线程运行间隔;

**memoryStoreEvictionPolicy**:缓存释放策略,LRU会优先清理最少使用的缓存；

**localTempSwap**：持久化策略，当堆内存或者非堆内存里面的元素已经满了的时候，将其中的元素临时的存放在磁盘上，重启后就会消失；

# 三、注解用法

```java
@Service
public class CacheService {

    private static final Logger LOGGER = LoggerFactory.getLogger(CacheService.class);

    @Resource
    private UserMapper userMapper ;

    @Cacheable(value="userEntity")  // 在缓存有效期内，首次查询才访问数据库
    public UserEntity getById (Integer id){
        // 通过日志，标识方法是否执行
        LOGGER.info("getById..."+id);
        return userMapper.selectById(id) ;
    }

    @CacheEvict(value="userEntity",key = "#id") //该ID数据更新，清空该ID缓存
    public void updateUser(Integer id) {
        UserEntity user = new UserEntity() ;
        user.setId(id);
        user.setUserName("myCache");
        userMapper.updateById(user);
    }
}
```

**@Cacheable**：注解标记在一个方法上，也可以标记在一个类上，标记在一个方法上表示该方法支持缓存，该方法被调用后将其返回值缓存起来，下次同样的请求参数执行该方法时可以直接从缓存中获取结果，而不需要再次执行该方法。

**@CacheEvict**：注解标记在需要清除缓存元素的方法或类上的，当标记在一个类上时表示其中所有的方法的执行都会触发缓存的清除操作，并且可以按照指定属性清除。

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware24-eht-cache