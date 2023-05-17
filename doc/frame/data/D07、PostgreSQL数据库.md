# 一、PostgreSQL简介

## 1、和MySQL的比较

PostgreSQL是一个功能强大的且开源关系型数据库系统，在网上PostgreSQL和MySQL一直有大量的对比分析。大多从性能，开源协议，SQL标准，开发难度等去比较，只要有比较就会有差距和差异，看看就好。

`絮叨一句`：编程世界里的对比是一直存在的，但是无论对比结果如何，当业务需要的时候，该用还是要用。MySQL和PostgreSQL对比很少占上风，但是MySQL在国内的使用依旧广泛。

## 2、PostgreSQL特性

- 多副本同步复制，满足金融级可靠性要求；
- 支持丰富的数据类型，除了常见基础的，还包括文本，图像，声音，视频，JSON等；
- 自带全文搜索功能，可以简化搜索功能实现流程；
- 高效处理图结构, 轻松实现"朋友的朋友的朋友"关系类型；
- 地理信息处理扩展，支持地图寻路相关业务；

# 二、开发环境整合

## 1、基础依赖

导入依赖包，版本会自动加载。本案例加载的是42.2.6版本号。

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

## 2、核心配置文件

这里使用Druid连接池管理。

```
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driverClassName: org.postgresql.Driver
      url: jdbc:postgresql://127.0.0.1:5432/db_01
      username: root01
      password: 123456
```

## 3、连接池配置

```java
@Configuration
public class DruidConfig {

    @Value("${spring.datasource.druid.url}")
    private String dbUrl;
    @Value("${spring.datasource.druid.username}")
    private String username;
    @Value("${spring.datasource.druid.password}")
    private String password;
    @Value("${spring.datasource.druid.driverClassName}")
    private String driverClassName;

    @Bean
    public DruidDataSource dataSource() {
        DruidDataSource datasource = new DruidDataSource();
        datasource.setUrl(dbUrl);
        datasource.setUsername(username);
        datasource.setPassword(password);
        datasource.setDriverClassName(driverClassName);
        return datasource;
    }
}
```

## 4、持久层配置

基于mybatis相关组件，在用法上和MySQL环境整合基本一致。

```
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  typeAliasesPackage: com.post.gresql.*.entity
  global-config:
    db-config:
      id-type: AUTO
      field-strategy: NOT_NULL
      logic-delete-value: -1
      logic-not-delete-value: 0
    banner: false
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    call-setters-on-nulls: true
    jdbc-type-for-null: 'null'
```

## 5、基础测试案例

提供一个数据查询，写入，分页查的基础使用案例。

```java
@Api(value = "UserController")
@RestController
public class UserController {

    @Resource
    private UserService userService ;

    @GetMapping("/selectById")
    public UserEntity selectById (Integer id){
        return userService.selectById(id) ;
    }

    @PostMapping("/insert")
    public Integer insert (UserEntity userEntity){
        return userService.insert(userEntity) ;
    }

    @GetMapping("/pageQuery")
    public PageInfo<UserEntity> pageQuery (@RequestParam("page") int page){
        int pageSize = 3 ;
        return userService.pageQuery(page,pageSize) ;
    }
}
```

# 三、JSON类型使用

PostgreSQL支持JSON数据类型格式，但是在用法上与一般数据类型有差异。

## 1、Json表字段创建

这里字段user_list为JSON类型，存储场景第一批用户有哪些，第二批用户有哪些，依次类推。

```sql
CREATE TABLE pq_user_json (
    ID INT NOT NULL,
    title VARCHAR (32) NOT NULL,
    user_list json NOT NULL,
    create_time TIMESTAMP (6) DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "user_json_pkey" PRIMARY KEY ("id")
);
```

## 2、类型转换器

定义一个数据库与实体对象的转换器，主要就是JSON数据和Java对象的转换。

```java
@MappedTypes({Object.class})
public class JsonTypeHandler extends BaseTypeHandler<Object> {

    private static final PGobject jsonObject = new PGobject();

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType) throws SQLException {
        jsonObject.setType("json");
        jsonObject.setValue(parameter.toString());
        ps.setObject(i, jsonObject);
    }

    @Override
    public Object getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return JSON.parseObject(rs.getString(columnName), Object.class);
    }

    @Override
    public Object getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return JSON.parseObject(rs.getString(columnIndex), Object.class);
    }

    @Override
    public Object getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return JSON.parseObject(cs.getString(columnIndex), Object.class);
    }
}
```

## 3、调用方法

指定字段的映射类型typeHandler即可。

```xml
<mapper namespace="com.post.gresql.mapper.UserJsonMapper" >

    <insert id="addUserJson" parameterType="com.post.gresql.entity.UserJsonEntity">
        INSERT INTO pq_user_json (id,title,user_list,create_time)
        VALUES (#{id}, #{title}, #{userList, typeHandler=com.post.gresql.config.JsonTypeHandler}, #{createTime})
    </insert>

</mapper>
```

## 4、JSON格式测试

JSON格式数据入库，出库查询。

```java
@RestController
public class UserJsonController {

    @Resource
    private UserJsonService userJsonService ;

    @GetMapping("/addUserJson")
    public boolean addUserJson (){
        List<UserEntity> userEntities = new ArrayList<>() ;
        UserEntity userEntity1 = new UserEntity(1,"LiSi",22,new Date());
        UserEntity userEntity2 = new UserEntity(2,"WangWu",23,new Date());
        userEntities.add(userEntity1);
        userEntities.add(userEntity2);
        UserJsonEntity userJsonEntity = new UserJsonEntity();
        userJsonEntity.setId(1);
        userJsonEntity.setTitle("第一批名单");
        userJsonEntity.setUserList(JSON.toJSONString(userEntities));
        userJsonEntity.setCreateTime(new Date());
        return userJsonService.addUserJson(userJsonEntity) ;
    }

    @GetMapping("/findUserJson")
    public List<UserEntity> findUserJson (@RequestParam("id") Integer id){
        UserJsonEntity userJsonEntity = userJsonService.findUserJson(id) ;
        return JSON.parseArray(userJsonEntity.getUserList(),UserEntity.class) ;
    }
}
```

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap01-data-source/case05-post-gresql