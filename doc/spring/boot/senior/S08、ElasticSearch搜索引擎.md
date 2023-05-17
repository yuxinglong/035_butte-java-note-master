# 一、安装和简介
ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。

# 二、与SpringBoot2整合

## 1、核心依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    <version>${spring-boot.version}</version>
</dependency>
```
## 2、配置文件

```yaml
spring:
  application:
    name: ware-elastic-search
  data:
    elasticsearch:
      # 默认 elasticsearch
      cluster-name: elasticsearch
      # 9200作为Http协议，主要用于外部通讯
      # 9300作为Tcp协议，jar之间就是通过tcp协议通讯
      cluster-nodes: 192.168.72.130:9300
```
## 3、实体类配置

Document配置，加上了@Document注解之后，默认情况下这个实体中所有的属性都会被建立索引、并且分词。

```
indexName索引名称 理解为数据库名 限定小写
type 理解为数据库的表名称
shards = 5 默认分区数
replicas = 1 每个分区默认的备份数
refreshInterval = "1s" 刷新间隔
indexStoreType = "fs"  索引文件存储类型
```

代码块

```java
@Document(indexName = "requestlogindex",type = "requestlog")
public class RequestLog {
    //Id注解Elasticsearch里相应于该列就是主键，查询时可以使用主键查询
    @Id
    private Long id;
    private String orderNo;
    private String userId;
    private String userName;
    private String createTime;
}
```

## 4、数据交互层

实现ElasticsearchRepository接口。

```java
public interface RequestLogRepository 
extends ElasticsearchRepository<RequestLog,Long> {
}
```

## 5、演示案例

数据增加，修改，查询，排序，多条件查询。

```java
@Service
public class RequestLogServiceImpl implements RequestLogService {
    @Resource
    private RequestLogRepository requestLogRepository ;
    @Override
    public String esInsert(Integer num) {
        for (int i = 0 ; i < num ; i++){
            RequestLog requestLog = new RequestLog() ;
            requestLog.setId(System.currentTimeMillis());
            requestLog.setOrderNo(DateUtil.formatDate(new Date(),DateUtil.DATE_FORMAT_02)+System.currentTimeMillis());
            requestLog.setUserId("userId"+i);
            requestLog.setUserName("张三"+i);
            requestLog.setCreateTime(DateUtil.formatDate(new Date(),DateUtil.DATE_FORMAT_01));
            requestLogRepository.save(requestLog) ;
        }
        return "success" ;
    }
    @Override
    public Iterable<RequestLog> esFindAll (){
        return requestLogRepository.findAll() ;
    }
    @Override
    public String esUpdateById(RequestLog requestLog) {
        requestLogRepository.save(requestLog);
        return "success" ;
    }
    @Override
    public Optional<RequestLog> esSelectById(Long id) {
        return requestLogRepository.findById(id) ;
    }
    @Override
    public Iterable<RequestLog> esFindOrder() {
        // 用户名倒序
        // Sort sort = new Sort(Sort.Direction.DESC,"userName.keyword") ;
        // 创建时间正序
        Sort sort = new Sort(Sort.Direction.ASC,"createTime.keyword") ;
        return requestLogRepository.findAll(sort) ;
    }
    @Override
    public Iterable<RequestLog> esFindOrders() {
        List<Sort.Order> sortList = new ArrayList<>() ;
        Sort.Order sort1 = new Sort.Order(Sort.Direction.ASC,"createTime.keyword") ;
        Sort.Order sort2 = new Sort.Order(Sort.Direction.DESC,"userName.keyword") ;
        sortList.add(sort1) ;
        sortList.add(sort2) ;
        Sort orders = Sort.by(sortList) ;
        return requestLogRepository.findAll(orders) ;
    }
    @Override
    public Iterable<RequestLog> search() {
        // 全文搜索关键字
        /*
        String queryString="张三";
        QueryStringQueryBuilder builder = new QueryStringQueryBuilder(queryString);
        requestLogRepository.search(builder) ;
        */
        /*
         * 多条件查询
         */
         QueryBuilder builder = QueryBuilders.boolQuery()
                // .must(QueryBuilders.matchQuery("userName.keyword", "历张")) 搜索不到
               .must(QueryBuilders.matchQuery("userName", "张三")) // 可以搜索
               .must(QueryBuilders.matchQuery("orderNo", "20190613736278243"));
        return requestLogRepository.search(builder) ;
    }
}
```

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware08-elastic-search