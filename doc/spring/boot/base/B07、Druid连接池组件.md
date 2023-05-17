# 一、Druid连接池

## 1、druid简介

Druid连接池是阿里巴巴开源的数据库连接池项目。Druid连接池为监控而生，内置强大的监控功能，监控特性不影响性能。功能强大，能防SQL注入，内置Loging能诊断Hack应用行为。

Druid连接池是阿里巴巴内部唯一使用的连接池，在内部数据库相关中间件TDDL/DRDS 都内置使用强依赖了Druid连接池，经过阿里内部数千上万的系统大规模验证，经过历年双十一超大规模并发验证。

## 2、druid特点

- 稳定性特性，阿里巴巴的业务验证
- 完备的监控信息，快速诊断系统的瓶颈
- 内置了WallFilter 提供防SQL注入功能

# 二、SpringBoot2整合

## 1、引入核心依赖

```xml
<!-- 数据库依赖 -->
<dependency>
    <groupid>mysql</groupid>
    <artifactid>mysql-connector-java</artifactid>
    <version>5.1.21</version>
</dependency>
<dependency>
    <groupid>com.alibaba</groupid>
    <artifactid>druid-spring-boot-starter</artifactid>
    <version>1.1.13</version>
</dependency>
<!-- JDBC 依赖 -->
<dependency>
    <groupid>org.springframework.boot</groupid>
    <artifactid>spring-boot-starter-jdbc</artifactid>
</dependency>
```

## 2、数据源配置文件

```yaml
spring:
  application:
    # 应用名称
    name: node07-boot-druid
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driverClassName: com.mysql.jdbc.Driver
      url: jdbc:mysql://localhost:3306/data_one?useUnicode=true&amp;characterEncoding=UTF8&amp;zeroDateTimeBehavior=convertToNull&amp;useSSL=false
      username: root
      password: 123
      initial-size: 10
      max-active: 100
      min-idle: 10
      max-wait: 60000
      pool-prepared-statements: true
      max-pool-prepared-statement-per-connection-size: 20
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      max-evictable-idle-time-millis: 60000
      validation-query: SELECT 1 FROM DUAL
      # validation-query-timeout: 5000
      test-on-borrow: false
      test-on-return: false
      test-while-idle: true
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
      #filters: #配置多个英文逗号分隔(统计，sql注入，log4j过滤)
      filters: stat,wall
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
```

## 3、核心配置类

```java
import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.support.http.StatViewServlet;
import com.alibaba.druid.support.http.WebStatFilter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;

/**
 * Druid数据库连接池配置文件
 */
@Configuration
public class DruidConfig {
    private static final Logger logger = LoggerFactory.getLogger(DruidConfig.class);
    @Value("${spring.datasource.druid.url}")
    private String dbUrl;
    @Value("${spring.datasource.druid.username}")
    private String username;
    @Value("${spring.datasource.druid.password}")
    private String password;
    @Value("${spring.datasource.druid.driverClassName}")
    private String driverClassName;
    @Value("${spring.datasource.druid.initial-size}")
    private int initialSize;
    @Value("${spring.datasource.druid.max-active}")
    private int maxActive;
    @Value("${spring.datasource.druid.min-idle}")
    private int minIdle;
    @Value("${spring.datasource.druid.max-wait}")
    private int maxWait;
    @Value("${spring.datasource.druid.pool-prepared-statements}")
    private boolean poolPreparedStatements;
    @Value("${spring.datasource.druid.max-pool-prepared-statement-per-connection-size}")
    private int maxPoolPreparedStatementPerConnectionSize;
    @Value("${spring.datasource.druid.time-between-eviction-runs-millis}")
    private int timeBetweenEvictionRunsMillis;
    @Value("${spring.datasource.druid.min-evictable-idle-time-millis}")
    private int minEvictableIdleTimeMillis;
    @Value("${spring.datasource.druid.max-evictable-idle-time-millis}")
    private int maxEvictableIdleTimeMillis;
    @Value("${spring.datasource.druid.validation-query}")
    private String validationQuery;
    @Value("${spring.datasource.druid.test-while-idle}")
    private boolean testWhileIdle;
    @Value("${spring.datasource.druid.test-on-borrow}")
    private boolean testOnBorrow;
    @Value("${spring.datasource.druid.test-on-return}")
    private boolean testOnReturn;
    @Value("${spring.datasource.druid.filters}")
    private String filters;
    @Value("{spring.datasource.druid.connection-properties}")
    private String connectionProperties;
    /**
     * Druid 连接池配置
     */
    @Bean     //声明其为Bean实例
    public DruidDataSource dataSource() {
        DruidDataSource datasource = new DruidDataSource();
        datasource.setUrl(dbUrl);
        datasource.setUsername(username);
        datasource.setPassword(password);
        datasource.setDriverClassName(driverClassName);
        datasource.setInitialSize(initialSize);
        datasource.setMinIdle(minIdle);
        datasource.setMaxActive(maxActive);
        datasource.setMaxWait(maxWait);
        datasource.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
        datasource.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
        datasource.setMaxEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
        datasource.setValidationQuery(validationQuery);
        datasource.setTestWhileIdle(testWhileIdle);
        datasource.setTestOnBorrow(testOnBorrow);
        datasource.setTestOnReturn(testOnReturn);
        datasource.setPoolPreparedStatements(poolPreparedStatements);
        datasource.setMaxPoolPreparedStatementPerConnectionSize(maxPoolPreparedStatementPerConnectionSize);
        try {
            datasource.setFilters(filters);
        } catch (Exception e) {
            logger.error("druid configuration initialization filter", e);
        }
        datasource.setConnectionProperties(connectionProperties);
        return datasource;
    }
    /**
     * JDBC操作配置
     */
    @Bean(name = "dataOneTemplate")
    public JdbcTemplate jdbcTemplate (@Autowired DruidDataSource dataSource){
        return new JdbcTemplate(dataSource) ;
    }

    /**
     * 配置 Druid 监控界面
     */
    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean srb =
                new ServletRegistrationBean(new StatViewServlet(),"/druid/*");
        //设置控制台管理用户
        srb.addInitParameter("loginUsername","root");
        srb.addInitParameter("loginPassword","root");
        //是否可以重置数据
        srb.addInitParameter("resetEnable","false");
        return srb;
    }
    @Bean
    public FilterRegistrationBean statFilter(){
        //创建过滤器
        FilterRegistrationBean frb =
                new FilterRegistrationBean(new WebStatFilter());
        //设置过滤器过滤路径
        frb.addUrlPatterns("/*");
        //忽略过滤的形式
        frb.addInitParameter("exclusions",
                "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        return frb;
    }
}
```

## 4、简单测试类

```java
@RestController
public class DruidController {
    private static final Logger LOG = LoggerFactory.getLogger(DruidController.class);
    @Resource
    private JdbcTemplate jdbcTemplate ;
    @RequestMapping("/druidData")
    public String druidData (){
        String sql = "SELECT COUNT(1) FROM d_phone" ;
        Integer countOne = jdbcTemplate.queryForObject(sql,Integer.class) ;
        // countOne==2
        LOG.info("countOne=="+countOne);
        return "success" ;
    }
}
```

# 三、测试效果

完成一次数据请求后，访问如下链接。

![输入图片说明](https://images.gitee.com/uploads/images/2022/0124/233138_644aeb80_5064118.jpeg)

```
http://localhost:8007/druid
输入配置的用户名和密码：
root root
```

## 1、Druid监控首页

主要展示链接数据库的基础信息。

![输入图片说明](https://images.gitee.com/uploads/images/2022/0124/233250_20672c52_5064118.jpeg )

## 2、Druid监控数据源

连接池配置的各项详细属性，可以参考这里查看，无需再从网上查找。

![输入图片说明](https://images.gitee.com/uploads/images/2022/0124/233319_461d1074_5064118.jpeg)

## 3、Druid监控SQL执行

所有执行的SQL，都会在这里被监控到，且会有SQL执行的详细计划。

![输入图片说明](https://images.gitee.com/uploads/images/2022/0124/233347_8e48c48b_5064118.jpeg)

**参考源码** ：https://gitee.com/cicadasmile/spring-boot-base/tree/master/node07-boot-druid