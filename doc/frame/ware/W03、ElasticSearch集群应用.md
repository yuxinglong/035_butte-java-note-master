# 一、集群环境搭建

## 1、环境概览

ES版本6.3.2，集群名称esmaster，虚拟机centos7。

| 服务群 | 角色划分 | 说明 |
| --- |--- | --- | 
| en-master | master | 主节点：esnode1 |
| en-node01 | slave | 从节点：esnode2 |
| en-node02 | slave | 从节点：esnode3 |

在真正海量数据的业务场景中，ElasticSearch搜索引擎都是需要集群化管理的，实时搜素几十亿的数据十分常见。

## 2、集群配置

**配置文件**

```
vim /opt/elasticsearch-6.3.2/config/elasticsearch.yml
```

![](https://images.gitee.com/uploads/images/2022/0212/152520_00ba2af6_5064118.jpeg "09-1.jpg")

**主节点配置**

```
# 集群主节点配置
cluster.name: esmaster
node.master: true

# 节点名称 
node.name: esnode1

# 开发访问
network.host: 0.0.0.0
```

**从节点配置**

注意这里两个从节点配置，node.name分别配置为esnode2和esnode3即可。

```
# 集群名称
cluster.name: esmaster

# 节点名称
node.name: esnode2

# 开发访问
network.host: 0.0.0.0

# 主节点IP
discovery.zen.ping.unicast.hosts: ["192.168.72.133"]
```

**内存权限**

```
vim /etc/sysctl.conf

# 添加内容
vm.max_map_count=262144

# 执行
sysctl -p
```

## 3、集群启动

添加esroot用户，并授权。

```
/opt/elasticsearch-6.3.2/bin/elasticsearch
```

单服务查看

```
ps -aux |grep elasticsearch
```

集群状态查看

```
http://localhost:9200/_cluster/health?pretty

{
  "cluster_name" : "esmaster",  # 集群名称
  "status" : "green",   # 绿：健康，黄：亚健康，红：病态
  "timed_out" : false,  # 是否超时
  "number_of_nodes" : 3, # 节点个数
}
```

# 二、集群模式测试

## 1、环境配置

**dev环境**

配置单个节点，选择任意单节点，进行数据写入测试。

```
spring:
  data:
    elasticsearch:
      # 集群名称
      cluster-name: esmaster
      # 单节点
      # cluster-nodes: en-master:9300
      # cluster-nodes: en-node01:9300
      cluster-nodes: en-node02:9300
```

**test环境**

链接集群环境，进行数据读取测试。

```
spring:
  data:
    elasticsearch:
      # 集群名称
      cluster-name: esmaster
      # 集群节点
      cluster-nodes: en-master:9300,en-node01:9300,en-node02:9300
```

当然所有的操作都可以基于单节点或者集群环境测试。

## 2、实例对象

基于注解管理数据对象实例。

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;

@Document(indexName = "usersearchindex",type = "usersearch")
public class UserSearch {

    //Id注解Elasticsearch里相应于该列就是主键，查询时可以使用主键查询
    @Id
    private Long id;
    private String userId;
    private String userName;
    private String sex;
}
```

## 3、操作案例

提供一个数据查询操作和数据写入操作。

```java
import com.esearch.cluster.entity.UserSearch;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;

@Service
public class UserSearchServiceImpl implements UserSearchService {

    @Resource
    private UserSearchRepository userSearchRepository ;

    @Override
    public String esInsert(Integer num) {
        for (int i = 0 ; i < num ; i++){
            UserSearch userSearch = new UserSearch() ;
            userSearch.setId(System.currentTimeMillis());
            userSearch.setUserId("Name"+i);
            userSearch.setUserName("ZSan"+i);
            userSearch.setSex("Male"+i);
            userSearchRepository.save(userSearch) ;
        }
        return "success" ;
    }

    @Override
    public Iterable<UserSearch> esFindAll (){
        return userSearchRepository.findAll() ;
    }

}
```

# 三、集群控制台

这里是基于Kibana组件做的集群控制台。

## 1、数据列表

在discover面板中可以查看列表数据，也可以继续搜索。

**列表查询**

![](https://images.gitee.com/uploads/images/2022/0212/152552_f94d8205_5064118.jpeg "09-2.jpg")

**列表搜索**

![](https://images.gitee.com/uploads/images/2022/0212/152604_779cd204_5064118.jpeg "09-3.jpg")

## 2、开发工具

在dev_tools面板中可以执行ElasticSearch相关命令。

**查看集群健康状态**

```
GET /_cat/health?v
```

![](https://images.gitee.com/uploads/images/2022/0212/152622_42085506_5064118.jpeg "09-4.jpg")

**查询全部数据**

```
GET _search
{
  "query": {
    "match_all": {}
  }
}
```

![](https://images.gitee.com/uploads/images/2022/0212/152633_dfd68661_5064118.jpeg "09-5.jpg")

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap01-data-source/case09-esearch-cluster