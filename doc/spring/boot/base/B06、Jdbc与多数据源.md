# 一、JdbcTemplate对象

## 1、JdbcTemplate简介

在Spring Boot2.0框架下配置数据源和通过JdbcTemplate访问数据库的案例。

SpringBoot对数据库的操作在jdbc上面做了深层次的封装，使用spring的注入功能，可以把DataSource注册到JdbcTemplate之中。

## 2、JdbcTemplate核心方法

- execute方法：可以用于执行任何SQL语句；
- update方法batchUpdate方法：update方法用于执行新增、修改、删除等语句；batchUpdate方法用于执行批处理相关语句；
- query方法及queryFor方法：用于执行查询相关语句；
- call方法：用于执行存储过程、函数相关语句。

# 二、SpringBoot2应用

## 1、导入Jar包

```
<!-- 数据库依赖 -->
<dependency>
    <groupid>mysql</groupid>
    <artifactid>mysql-connector-java</artifactid>
    <version>5.1.21</version>
</dependency>
<!-- JDBC 依赖 -->
<dependency>
    <groupid>org.springframework.boot</groupid>
    <artifactid>spring-boot-starter-jdbc</artifactid>
</dependency>
```

## 2、配置数据源信息

```
spring:
  application:
    # 应用名称
    name: node06-boot-jdbc
  datasource:
    # 数据源一：data_one 库
    primary:
      # 2.0开始的版本必须这样配置
      jdbc-url: jdbc:mysql://localhost:3306/data_one
      #url: jdbc:mysql://localhost:3306/data_one
      username: root
      password: 123
      driver-class-name: com.mysql.jdbc.Driver
    # 数据源二：data_two 库
    secondary:
      # 2.0开始的版本必须这样配置
      jdbc-url: jdbc:mysql://localhost:3306/data_two
      #url: jdbc:mysql://localhost:3306/data_two
      username: root
      password: 123
      driver-class-name: com.mysql.jdbc.Driver
```

## 3、数据源代码配置

 **数据源一的配置** 

@Primary 注解表示该数据源作为默认的主数据库。

```java
/**
 * 数据源一配置
 */
@Configuration
public class DataOneConfig {

    @Primary    // 主数据库
    @Bean(name = "primaryDataSource")
    @Qualifier("primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource (){
        return DataSourceBuilder.create().build() ;
    }

    @Bean(name = "primaryJdbcTemplate")
    public JdbcTemplate primaryJdbcTemplate (
            @Qualifier("primaryDataSource") DataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
}
```

**数据源二配置** 

```java
/**
 * 数据源二配置
 */
@Configuration
public class DataTwoConfig {
    @Bean(name = "secondaryDataSource")
    @Qualifier("secondaryDataSource")
    @ConfigurationProperties(prefix="spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryJdbcTemplate")
    public JdbcTemplate secondaryJdbcTemplate(
            @Qualifier("secondaryDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

## 4、编写一个简单的测试类

```java
@RestController
public class JdbcController {
    private static final Logger LOG = LoggerFactory.getLogger(JdbcController.class);

    // 数据源一
    @Autowired
    @Qualifier("primaryJdbcTemplate")
    private JdbcTemplate primaryJdbcTemplate ;

    // 数据源二
    @Autowired
    @Qualifier("secondaryJdbcTemplate")
    private JdbcTemplate secondaryJdbcTemplate ;

    /**
     * 多数据源查询
     */
    @RequestMapping("/queryData")
    public String queryData (){
        String sql = "SELECT COUNT(1) FROM d_phone" ;
        Integer countOne = primaryJdbcTemplate.queryForObject(sql,Integer.class) ;
        Integer countTwo = secondaryJdbcTemplate.queryForObject(sql,Integer.class) ;
        LOG.info("countOne=="+countOne+";;countTwo=="+countTwo);
        return "SUCCESS" ;
    }
}
```

**参考源码** ：https://gitee.com/cicadasmile/spring-boot-base/tree/master/node06-boot-jdbc