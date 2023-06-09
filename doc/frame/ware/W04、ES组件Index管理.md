# 一、基础API简介

## 1、RestHighLevelClient

RestHighLevelClient的API作为ElasticSearch备受推荐的客户端组件，其封装系统操作ES的方法，包括索引结构管理，数据增删改查管理，常用查询方法，并且可以结合原生ES查询原生语法，功能十分强大。

![](https://images.gitee.com/uploads/images/2022/0212/153351_5f3243d1_5064118.png "01-1.png")

在使用RestHighLevelClient的语法时，通常涉及上面几个方面，在掌握基础用法之上可以根据业务特点进行一些自定义封装，这样可以更优雅的解决业务需求。

## 2、核心依赖

使用RestHighLevelClient需要依赖`rest-high-level-client`包，和ES相关基础依赖。

```xml
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-client</artifactId>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```

# 二、索引管理

这里不做过多描述，注意一点：因为ES的数据结构特点，所以不需要索引更新方法，新的字段在更新数据时直接写入即可，不需要提前更新索引结构。

```java
@Service
public class EsIndexOperation {

    @Resource
    private RestHighLevelClient client ;
    private final RequestOptions options = RequestOptions.DEFAULT;

    /**
     * 判断索引是否存在
     */
    public boolean checkIndex (String index) {
        try {
            return client.indices().exists(new GetIndexRequest(index), options);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return Boolean.FALSE ;
    }

    /**
     * 创建索引
     */
    public boolean createIndex (String indexName ,Map<String, Object> columnMap){
        try {
            if(!checkIndex(indexName)){
                CreateIndexRequest request = new CreateIndexRequest(indexName);
                if (columnMap != null && columnMap.size()>0) {
                    Map<String, Object> source = new HashMap<>();
                    source.put("properties", columnMap);
                    request.mapping(source);
                }
                this.client.indices().create(request, options);
                return Boolean.TRUE ;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return Boolean.FALSE;
    }

    /**
     * 删除索引
     */
    public boolean deleteIndex(String indexName) {
        try {
            if(checkIndex(indexName)){
                DeleteIndexRequest request = new DeleteIndexRequest(indexName);
                AcknowledgedResponse response = client.indices().delete(request, options);
                return response.isAcknowledged();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return Boolean.FALSE;
    }
}
```

# 三、数据管理

这里在更新数据时，可以直接修改索引结构，在dataMap中放进新的字段即可。

```java
@Service
public class EsDataOperation {

    @Resource
    private RestHighLevelClient client ;
    private final RequestOptions options = RequestOptions.DEFAULT;

    /**
     * 写入数据
     */
    public boolean insert (String indexName, Map<String,Object> dataMap){
        try {
            BulkRequest request = new BulkRequest();
            request.add(new IndexRequest(indexName,"doc").id(dataMap.remove("id").toString())
                    .opType("create").source(dataMap,XContentType.JSON));
            this.client.bulk(request, options);
            return Boolean.TRUE ;
        } catch (Exception e){
            e.printStackTrace();
        }
        return Boolean.FALSE;
    }

    /**
     * 批量写入数据
     */
    public boolean batchInsert (String indexName, List<Map<String,Object>> userIndexList){
        try {
            BulkRequest request = new BulkRequest();
            for (Map<String,Object> dataMap:userIndexList){
                request.add(new IndexRequest(indexName,"doc").id(dataMap.remove("id").toString())
                        .opType("create").source(dataMap,XContentType.JSON));
            }
            this.client.bulk(request, options);
            return Boolean.TRUE ;
        } catch (Exception e){
            e.printStackTrace();
        }
        return Boolean.FALSE;
    }

    /**
     * 更新数据，可以直接修改索引结构
     */
    public boolean update (String indexName, Map<String,Object> dataMap){
        try {
            UpdateRequest updateRequest = new UpdateRequest(indexName,"doc", dataMap.remove("id").toString());
            updateRequest.setRefreshPolicy(WriteRequest.RefreshPolicy.IMMEDIATE);
            updateRequest.doc(dataMap) ;
            this.client.update(updateRequest, options);
            return Boolean.TRUE ;
        } catch (Exception e){
            e.printStackTrace();
        }
        return Boolean.FALSE;
    }

    /**
     * 删除数据
     */
    public boolean delete (String indexName, String id){
        try {
            DeleteRequest deleteRequest = new DeleteRequest(indexName,"doc", id);
            this.client.delete(deleteRequest, options);
            return Boolean.TRUE ;
        } catch (Exception e){
            e.printStackTrace();
        }
        return Boolean.FALSE;
    }
}
```

# 四、查询操作

注意：查询总数的CountRequest语法，SearchRequest查询结果中数据转换语法，分页查询中需要指定偏移位置和分页大小。

```java
@Service
public class EsQueryOperation {

    @Resource
    private RestHighLevelClient client ;
    private final RequestOptions options = RequestOptions.DEFAULT;

    /**
     * 查询总数
     */
    public Long count (String indexName){
        // 指定创建时间
        BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery();
        queryBuilder.must(QueryBuilders.termQuery("createTime", 1611378102795L));

        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(queryBuilder);

        CountRequest countRequest = new CountRequest(indexName);
        countRequest.source(sourceBuilder);
        try {
            CountResponse countResponse = client.count(countRequest, options);
            return countResponse.getCount();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0L;
    }

    /**
     * 查询集合
     */
    public List<Map<String,Object>> list (String indexName) {
        // 查询条件,指定时间并过滤指定字段值
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery();
        queryBuilder.must(QueryBuilders.termQuery("createTime", 1611378102795L));
        queryBuilder.mustNot(QueryBuilders.termQuery("name","北京-李四"));
        sourceBuilder.query(queryBuilder);
        SearchRequest searchRequest = new SearchRequest(indexName);
        searchRequest.source(sourceBuilder);
        try {
            SearchResponse searchResp = client.search(searchRequest, options);
            List<Map<String,Object>> data = new ArrayList<>() ;
            SearchHit[] searchHitArr = searchResp.getHits().getHits();
            for (SearchHit searchHit:searchHitArr){
                Map<String,Object> temp = searchHit.getSourceAsMap();
                temp.put("id",searchHit.getId()) ;
                data.add(temp);
            }
            return data;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null ;
    }

    /**
     * 分页查询
     */
    public List<Map<String,Object>> page (String indexName,Integer offset,Integer size) {
        // 查询条件,指定时间并过滤指定字段值
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.from(offset);
        sourceBuilder.size(size);
        sourceBuilder.sort("createTime", SortOrder.DESC);
        SearchRequest searchRequest = new SearchRequest(indexName);
        searchRequest.source(sourceBuilder);
        try {
            SearchResponse searchResp = client.search(searchRequest, options);
            List<Map<String,Object>> data = new ArrayList<>() ;
            SearchHit[] searchHitArr = searchResp.getHits().getHits();
            for (SearchHit searchHit:searchHitArr){
                Map<String,Object> temp = searchHit.getSourceAsMap();
                temp.put("id",searchHit.getId()) ;
                data.add(temp);
            }
            return data;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null ;
    }
}
```

# 五、排序方式

排序除了常规的指定字段升序降序规则之外，还可以基于原生的脚本语法，基于自定义规则排序让一些特定的数据沉底或者置顶。

```java
@Service
public class EsSortOperation {

    @Resource
    private RestHighLevelClient client ;
    private final RequestOptions options = RequestOptions.DEFAULT;

    /**
     * 排序规则
     */
    public List<Map<String,Object>> sort (String indexName) {
        // 先升序时间，在倒序年龄
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.sort("createTime", SortOrder.ASC);
        sourceBuilder.sort("age",SortOrder.DESC) ;
        SearchRequest searchRequest = new SearchRequest(indexName);
        searchRequest.source(sourceBuilder);
        try {
            SearchResponse searchResp = client.search(searchRequest, options);
            List<Map<String,Object>> data = new ArrayList<>() ;
            SearchHit[] searchHitArr = searchResp.getHits().getHits();
            for (SearchHit searchHit:searchHitArr){
                Map<String,Object> temp = searchHit.getSourceAsMap();
                temp.put("id",searchHit.getId()) ;
                data.add(temp);
            }
            return data;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null ;
    }

    /**
     * 自定义排序规则
     */
    public List<Map<String,Object>> defSort (String indexName) {
        // 指定置换顺序的规则
        // [age 12-->60]\[age 19-->10]\[age 13-->30]\[age 18-->40],age其他值忽略为1
        Script script = new Script("def _ageSort = doc['age'].value == 12?60:" +
                                                            "(doc['age'].value == 19?10:" +
                                                            "(doc['age'].value == 13?30:" +
                                                            "(doc['age'].value == 18?40:1)));" + "_ageSort;");
        ScriptSortBuilder sortBuilder = SortBuilders.scriptSort(script,ScriptSortBuilder.ScriptSortType.NUMBER);
        sortBuilder.order(SortOrder.ASC);
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.sort(sortBuilder);
        SearchRequest searchRequest = new SearchRequest(indexName);
        searchRequest.source(sourceBuilder);
        try {
            SearchResponse searchResp = client.search(searchRequest, options);
            List<Map<String,Object>> data = new ArrayList<>() ;
            SearchHit[] searchHitArr = searchResp.getHits().getHits();
            for (SearchHit searchHit:searchHitArr){
                Map<String,Object> temp = searchHit.getSourceAsMap();
                temp.put("id",searchHit.getId()) ;
                data.add(temp);
            }
            return data;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null ;
    }
}
```

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap05-middle-ware/soft01-elastic-search