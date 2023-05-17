# 一、数据同步简介

## 1、场景描述

如果经常接触数据开发，会有这样一个场景，服务A提供一个数据源，假设称为动态数据源A，需要读取该数据源下的数据；服务B提供一个数据源，假设称为动态数据源B，需要写入数据到该数据源。这个场景通常描述为数据同步，或者数据搬运。

## 2、基本流程

![](https://images.gitee.com/uploads/images/2022/0212/140440_59c0ac85_5064118.png "03-1.png")

 基于上述流程图，整体步骤如下：
 - 测试多个数据源是否连接成功，并动态管理;
 - 判断数据源提供的账号是否有操作权限，例如读写;
 - 读取数据源A的表结构，在数据源B创建表;
 - 数据读取或者分页读取，写入数据源B中;
 - 在不知道表结构情况下，还需要读取表结构，生成SQL;

## 3、JDBC基础API

- Statement

Java中JDBC下执行数据库操作的一个重要接口，在已经建立数据库连接的基础上，向数据库发送要执行的SQL语句。

- PreparedStatement

继承Statement接口，且实现SQL预编译，可以提高批量处理效率。常应用于批量数据写入场景。

- ResultSet

存储JDBC查询结果集的对象，ResultSet接口提供从当前行检索列值的方法。

# 二、基础工具封装

## 1、数据源管理

提供一个数据源管理的Factory，当前场景下主要管理一个读库即数据源A，和一个写库即数据源B，数据源连接验证通过，放入容器中。

```java
@Component
public class ConnectionFactory {

    private volatile Map<String, Connection> connectionMap = new HashMap<>();

    @Resource
    private JdbcConfig jdbcConfig ;

    @PostConstruct
    public void init (){
        ConnectionEntity read = new ConnectionEntity(
        "MySql","jdbc:mysql://localhost:3306/data_read","user01","123");
        if (jdbcConfig.getConnection(read) != null){
            connectionMap.put(JdbcConstant.READ,jdbcConfig.getConnection(read));
        }
        ConnectionEntity write = new ConnectionEntity(
        "MySql","jdbc:mysql://localhost:3306/data_write","user01","123");
        if (jdbcConfig.getConnection(write) != null){
            connectionMap.put(JdbcConstant.WRITE,jdbcConfig.getConnection(write));
        }
    }

    public Connection getByKey (final String key){
        return connectionMap.get(key) ;
    }
}
```

## 2、动态SQL拼接

**基础SQL管理**

主要提供SQL的基础模板，例如全表查，分页查，表结构查询。

```java
public class BaseSql {
    public static String READ_SQL = "SELECT * FROM %s LIMIT 1";
    public static String WRITE_SQL = "INSERT INTO %s (SELECT * FROM %s WHERE 1=0)" ;
    public static String CREATE_SQL = "SHOW CREATE TABLE %s" ;
    public static String SELECT_SQL = "SELECT * FROM %s" ;
    public static String COUNT_SQL = "SELECT COUNT(1) countNum FROM %s" ;
    public static String PAGE_SQL = "SELECT * FROM %s LIMIT %s,%s" ;
    public static String STRUCT_SQL (){
        StringBuffer sql = new StringBuffer() ;
        sql.append(" SELECT                     ");
        sql.append("     COLUMN_NAME,           ");
        sql.append("     IS_NULLABLE,           ");
        sql.append("     COLUMN_TYPE,           ");
        sql.append("     COLUMN_KEY,            ");
        sql.append("     COLUMN_COMMENT         ");
        sql.append(" FROM                       ");
        sql.append(" information_schema.COLUMNS ");
        sql.append(" WHERE                      ");
        sql.append(" table_schema = '%s'        ");
        sql.append(" AND table_name = '%s'      ");
        return String.valueOf(sql) ;
    }
}
```

**SQL参数拼接**

根据SQL模板中缺失的参数，进行动态补全，生成完成SQL语句。

```java
public class BuildSql {
    /**
     * 读权限SQL
     */
    public static String buildReadSql(String table) {
        String readSql = null ;
        if (StringUtils.isNotEmpty(table)){
            readSql = String.format(BaseSql.READ_SQL, table);
        }
        return readSql;
    }
    /**
     * 读权限SQL
     */
    public static String buildWriteSql(String table){
        String writeSql = null ;
        if (StringUtils.isNotEmpty(table)){
            writeSql = String.format(BaseSql.WRITE_SQL, table,table);
        }
        return writeSql ;
    }
    /**
     * 表创建SQL
     */
    public static String buildStructSql (String table){
        String structSql = null ;
        if (StringUtils.isNotEmpty(table)){
            structSql = String.format(BaseSql.CREATE_SQL, table);
        }
        return structSql ;
    }
    /**
     * 表结构SQL
     */
    public static String buildTableSql (String schema,String table){
        String structSql = null ;
        if (StringUtils.isNotEmpty(table)){
            structSql = String.format(BaseSql.STRUCT_SQL(), schema,table);
        }
        return structSql ;
    }
    /**
     * 全表查询SQL
     */
    public static String buildSelectSql (String table){
        String selectSql = null ;
        if (StringUtils.isNotEmpty(table)){
            selectSql = String.format(BaseSql.SELECT_SQL,table);
        }
        return selectSql ;
    }
    /**
     * 总数查询SQL
     */
    public static String buildCountSql (String table){
        String countSql = null ;
        if (StringUtils.isNotEmpty(table)){
            countSql = String.format(BaseSql.COUNT_SQL,table);
        }
        return countSql ;
    }
    /**
     * 分页查询SQL
     */
    public static String buildPageSql (String table,int offset,int size){
        String pageSql = null ;
        if (StringUtils.isNotEmpty(table)){
            pageSql = String.format(BaseSql.PAGE_SQL,table,offset,size);
        }
        return pageSql ;
    }
}
```

# 三、业务化流程

## 1、基础鉴权

读库尝试一次单条数据读取，写库尝试一次不成立条件的写入，如果没有权限，会抛出相应异常。

```java
@RestController
public class CheckController {
    @Resource
    private ConnectionFactory connectionFactory ;
    // MySQLSyntaxErrorException: SELECT command denied to user
    @GetMapping("/checkRead")
    public String checkRead (){
        try {
            String sql = BuildSql.buildReadSql("rw_read") ;
            ExecuteSqlUtil.query(connectionFactory.getByKey(JdbcConstant.READ),sql) ;
            return "success" ;
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return "fail" ;
    }
    // MySQLSyntaxErrorException: INSERT command denied to user
    @GetMapping("/checkWrite")
    public String checkWrite (){
        try {
            String sql = BuildSql.buildWriteSql("rw_read") ;
            ExecuteSqlUtil.update(connectionFactory.getByKey(JdbcConstant.WRITE),sql) ;
            return "success" ;
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return "fail" ;
    }
}
```

## 2、同步表结构

这里执行最简单操作，把读库表创建语句查询出来，丢到写库中执行。

```java
@RestController
public class StructController {
    @Resource
    private ConnectionFactory connectionFactory ;
    @GetMapping("/syncStruct")
    public String syncStruct (){
        try {
            String sql = BuildSql.buildStructSql("rw_read") ;
            ResultSet resultSet = ExecuteSqlUtil.query(connectionFactory.getByKey(JdbcConstant.READ),sql) ;
            String createTableSql = null ;
            while (resultSet.next()){
                createTableSql = resultSet.getString("Create Table") ;
            }
            if (StringUtils.isNotEmpty(createTableSql)){
                ExecuteSqlUtil.update(connectionFactory.getByKey(JdbcConstant.WRITE),createTableSql) ;
            }
            return "success" ;
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return "fail" ;
    }
}
```

## 3、同步表数据

读库的表数据读取，批量放入写库中。这里特别说一个方法：statement.setObject();在不知道参数个数和类型时，自动适配数据类型。

```java
@RestController
public class DataSyncController {
    @Resource
    private ConnectionFactory connectionFactory ;
    @GetMapping("/dataSync")
    public List<RwReadEntity> dataSync (){
        List<RwReadEntity> rwReadEntities = new ArrayList<>() ;
        try {
            Connection readConnection = connectionFactory.getByKey(JdbcConstant.READ) ;
            String sql = BuildSql.buildSelectSql("rw_read") ;
            ResultSet resultSet = ExecuteSqlUtil.query(readConnection,sql) ;
            while (resultSet.next()){
                RwReadEntity rwReadEntity = new RwReadEntity() ;
                rwReadEntity.setId(resultSet.getInt("id"));
                rwReadEntity.setSign(resultSet.getString("sign"));
                rwReadEntities.add(rwReadEntity) ;
            }
            if (rwReadEntities.size() > 0){
                Connection writeConnection = connectionFactory.getByKey(JdbcConstant.WRITE) ;
                writeConnection.setAutoCommit(false);
                PreparedStatement statement = writeConnection.prepareStatement("INSERT INTO rw_read VALUES(?,?)");
                // 基于动态获取列，和statement.setObject();自动适配数据类型
                for (int i = 0 ; i < rwReadEntities.size() ; i++){
                    RwReadEntity rwReadEntity = rwReadEntities.get(i) ;
                    statement.setInt(1,rwReadEntity.getId()) ;
                    statement.setString(2,rwReadEntity.getSign()) ;
                    statement.addBatch();
                    if (i>0 && i%2==0){
                        statement.executeBatch() ;
                    }
                }
                // 处理最后一批数据
                statement.executeBatch();
                writeConnection.commit();
            }
            return rwReadEntities ;
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return null ;
    }
}
```

## 4、分页查询

提供一个分页查询工具，在数据量大的情况下不能一次性读取大量的数据，避免资源占用过高。

```java
public class PageUtilEntity {
    /**
     * 分页生成方法
     */
    public static PageHelperEntity<Object> pageResult (int total, int pageSize,int currentPage, List dataList){
        PageHelperEntity<Object> pageBean = new PageHelperEntity<Object>();
        // 总页数
        int totalPage = PageHelperEntity.countTotalPage(pageSize,total) ;
        // 分页列表
        List<Integer> pageList = PageHelperEntity.pageList(currentPage,pageSize,total) ;
        // 上一页
        int prevPage = 0 ;
        if (currentPage==1){
            prevPage = currentPage ;
        } else if (currentPage>1&&currentPage<=totalPage){
            prevPage = currentPage -1 ;
        }
        // 下一页
        int nextPage =0 ;
        if (totalPage==1){
            nextPage = currentPage ;
        } else if (currentPage<=totalPage-1){
            nextPage = currentPage+1 ;
        }
        pageBean.setDataList(dataList);
        pageBean.setTotal(total);
        pageBean.setPageSize(pageSize);
        pageBean.setCurrentPage(currentPage);
        pageBean.setTotalPage(totalPage);
        pageBean.setPageList(pageList);
        pageBean.setPrevPage(prevPage);
        pageBean.setNextPage(nextPage);
        pageBean.initjudge();
        return  pageBean ;
    }
}
```

# 四、最后总结

很多复杂度偏高的业务，越是需要借助基础API解决，因为复杂度高，不容易抽象化统一封装，如果数据同步这块业务，可以适配多种数据库，完全可以独立封装为中间件，开源项目中关于多方数据同步或计算的中间件也有好多，可以自行了解下，增长眼界开阔思路。

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap01-data-source/case03-read-write