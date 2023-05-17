# 一、关系型数据源

## 1、动态数据源

![](https://images.gitee.com/uploads/images/2022/0212/140143_d5c27123_5064118.png "02-1.png")

动态管理数据源的基本功能：数据源加载，容器维护，持久化管理。

## 2、关系型数据库

不同厂商的关系型数据库，提供的链接方式，驱动包，驱动类名都是不一样的，Java数据库连接API，JDBC是Java语言中用来规范客户端程序如何来访问数据库的应用程序接口，提供了诸如查询和更新数据库中数据的方法，且适配大部分关系型数据库。

## 3、适配要素

核心要素：驱动包、驱动类名、URL格式、默认端口。

关系型数据库很多，这里一定是不全的，根据需要自行完善即可。

```java
public enum DataSourceType {

    MySql("MySql", "com.mysql.jdbc.Driver"),
    Oracle("Oracle", "oracle.jdbc.OracleDriver"),
    DB2("DB2", "com.ibm.db2.jcc.DB2Driver");

    private String dataSourceName;
    private String driverClassName;

    public static String getDriver (String dataSourceName) {
        DataSourceType[] types = DataSourceType.values();
        for (DataSourceType type : types) {
            if (type.getDataSourceName().equals(dataSourceName)) {
                return type.getDriverClassName();
            }
        }
        return null;
    }
    DataSourceType (String dataSourceName,String driverClassName){
        this.dataSourceName = dataSourceName ;
        this.driverClassName = driverClassName ;
    }
}
```

## 4、JDBC基础API

**DriverManager**

管理JDBC驱动程序的基本服务API。调用方法Class.forName，显式地加载驱动程序类，正好适用于动态数据源的业务场景，数据源类型未知情况。加载Driver类并在DriverManager类注册后，即可用来与数据库建立连接。

**DataSource**

DataSource接口，由驱动程序供应商实现，负责建立与数据库的连接，当在应用程序中访问数据库时，常用于获取操作数据的Connection对象。

**Connection**

Connection接口代表与特定的数据库的连接，要对数据库数据进行操作,首先要获取数据库连接，Connection实现就像在应用程序中与数据库之间开通了一条通道，通过DriverManager类或DataSource类都可获取Connection实例。

# 二、链接和管理

这里几个核心类的封装思路：模块化功能，API分开封装，如果需要适配处理各类数据源类型，则分别可以向上抽象提取，向下自定义适配策略，设计模式影响下的基本意识。

## 1、链接工具

基于DriverManager管理数据源的驱动加载，链接获取等。

```java
public class ConnectionUtil {

    public static synchronized Connection getConnect(String driverClassName,String userName,
                                                  String passWord,String jdbcUrl) {
        Properties prop = new Properties();
        prop.put("user", userName);
        prop.put("password", passWord);
        return connect(driverClassName,prop,jdbcUrl) ;
    }

    private static synchronized Connection connect(String driverClassName,
                                                   Properties prop,String jdbcUrl) {
        try {
            Class.forName(driverClassName);
            DriverManager.setLoginTimeout(JdbcConstant.LOGIN_TIMEOUT);
            return DriverManager.getConnection(jdbcUrl, prop);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null ;
    }

}
```

## 2、API工具类

提供API配置获取类，加载需要的数据源API，关闭资源等基本操作。

```java
@Component
public class JdbcConfig {

    /**
     * 获取数据源连接
     */
    public Connection getConnection (ConnectionEntity connectionEntity){
        String dataTypeName = connectionEntity.getDataTypeName();
        String driverClassName = DataSourceType.getDriver(dataTypeName) ;
        if (driverClassName == null){
            throw new RuntimeException("不支持该数据源类型") ;
        }
        connectionEntity.setDriverClassName(driverClassName);
        return ConnectionUtil.getConnect(connectionEntity.getDriverClassName(),
                connectionEntity.getUserName(),
                connectionEntity.getPassWord(),
                connectionEntity.getJdbcUrl()) ;
    }

}
```

## 3、数据源容器

维护一个Map容器，管理数据源的添加，删除，动态获取等基本需求。

```java
@Component
public class DataSourceFactory {

    private volatile Map<Integer, DataSource> dataSourceMap = new HashMap<>();

    @Resource
    private JdbcConfig jdbcConfig ;
    @Resource
    private ConnectionMapper connectionMapper ;

    /**
     * 数据源API包装
     */
    private static DataSource getDataSource (ConnectionEntity connectionEntity){
        DruidDataSource datasource = new DruidDataSource();
        datasource.setUrl(connectionEntity.getJdbcUrl());
        datasource.setUsername(connectionEntity.getUserName());
        datasource.setPassword(connectionEntity.getPassWord());
        datasource.setDriverClassName(connectionEntity.getDriverClassName());
        return datasource ;
    }

    /**
     * 获取 JDBC 链接
     */
    public JdbcTemplate getById (Integer id){
        return new JdbcTemplate(dataSourceMap.get(id)) ;
    }

    /**
     * 移除 数据源
     */
    public void removeById (Integer id) {
        dataSourceMap.remove(id) ;
    }

    /**
     * 添加数据源管理
     * 注意这里的方法，连接验证之后直接调用
     */
    public void addDataSource (ConnectionEntity connectionEntity){
        DataSource dataSource = getDataSource(connectionEntity);
        dataSourceMap.put(connectionEntity.getId(),dataSource) ;
    }
}
```

## 4、流程测试

基于动态的数据源，查询表数据，这里操作的表示已知的表结构，实际上动态数据源的表结构都是需要再次动态获取表字段，才能操作。(下节数据动态读取和写入会详说)

```java
@Api(value = "JdbcQueryController")
@RestController
public class JdbcQueryController {

    @Resource
    private DataSourceFactory dataSourceFactory ;

    @GetMapping("getList")
    public List<ConnectionEntity> getList (@RequestParam("id") Integer id){
        String sql = "SELECT * FROM jm_connection WHERE state='1'" ;
        JdbcTemplate jdbcTemplate = dataSourceFactory.getById(id);
        List<ConnectionEntity> connectionEntities = jdbcTemplate.query(sql,
                new BeanPropertyRowMapper<>(ConnectionEntity.class));
        return connectionEntities ;
    }
}
```

# 三、批量管理

持久化数据源的配置信息，多了一步配置信息入库，和入库信息加载到容器，使用时动态获取。

## 1、库表Mapper结构

存储配置信息的表结构，转换Mapper文件。

```xml
<mapper namespace="com.dynamic.add.mapper.ConnectionMapper">
    <!-- 通用查询映射结果 -->
    <resultMap id="BaseResultMap" type="com.dynamic.add.entity.ConnectionEntity">
        <id column="id" property="id" />
        <result column="data_type_name" property="dataTypeName" />
        <result column="driver_class_name" property="driverClassName" />
        <result column="jdbc_url" property="jdbcUrl" />
        <result column="user_name" property="userName" />
        <result column="pass_word" property="passWord" />
        <result column="create_time" property="createTime" />
        <result column="update_time" property="updateTime" />
        <result column="state" property="state" />
    </resultMap>
    <select id="getAllList" resultMap="BaseResultMap" >
        SELECT * FROM jm_connection WHERE state='1'
    </select>
</mapper>
```

## 2、持久化管理

测试数据源链接是否成功，可用的数据源链接，配置信息入库保存。

```java
@Service
public class ConnectionServiceImpl implements ConnectionService {

    @Resource
    private ConnectionMapper connectionMapper ;
    @Resource
    private JdbcConfig jdbcConfig ;
    @Resource
    private DataSourceFactory dataSourceFactory ;

    @Override
    public boolean testConnection(ConnectionEntity connectionEntity) {
        return jdbcConfig.getConnection(connectionEntity) !=null ;
    }

    @Override
    public boolean addConnection(ConnectionEntity connectionEntity) {
        Connection connection = jdbcConfig.getConnection(connectionEntity) ;
        if (connection !=null){
            int addFlag = connectionMapper.insert(connectionEntity);
            if (addFlag > 0){
                dataSourceFactory.addDataSource(connectionEntity) ;
                return true ;
            }
        }
        return false ;
    }
}
```

## 3、动态加载

容器工厂类中，添加一个初始化的方法，加载入库的数据源配置信息。

```java
@Component
public class DataSourceFactory {
    /**
     * 初始化 JDBC 链接API
     */
    @PostConstruct
    public void init (){
        List<ConnectionEntity> connectionList = connectionMapper.getAllList();
        if (connectionList != null && connectionList.size()>0){
            for (ConnectionEntity connectionEntity:connectionList) {
                Connection connection = jdbcConfig.getConnection(connectionEntity) ;
                if (connection != null){
                    DataSource dataSource = getDataSource(connectionEntity);
                    dataSourceMap.put(connectionEntity.getId(),dataSource) ;
                }
            }
        }
    }
}
```

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap01-data-source/case02-dynamic-add