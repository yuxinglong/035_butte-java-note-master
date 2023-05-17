# 一、配置详解

场景描述：MySQL数据表以全量和增量的方式向ElasticSearch搜索引擎同步。

## 1、下载内容

- elasticsearch 版本 6.3.2
- logstash 版本 6.3.2
- mysql-connector-java-5.1.13.jar

## 2、核心配置

- 路径：/usr/local/logstash
- 新建配置目录：sync-config

1)、配置全文

/usr/local/logstash/sync-config/cicadaes.conf

```
input {
    stdin {}
    jdbc {
        jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/cicada?characterEncoding=utf8"
        jdbc_user => "root"
        jdbc_password => "root123"
        jdbc_driver_library => "/usr/local/logstash/sync-config/mysql-connector-java-5.1.13.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_paging_enabled => "true"
        jdbc_page_size => "50000"
        jdbc_default_timezone => "Asia/Shanghai"
        statement_filepath => "/usr/local/logstash/sync-config/user_sql.sql"
        schedule => "* * * * *"
        type => "User"
        lowercase_column_names => false
        record_last_run => true
        use_column_value => true
        tracking_column => "updateTime"
        tracking_column_type => "timestamp"
        last_run_metadata_path => "/usr/local/logstash/sync-config/user_last_time"
        clean_run => false
    }
    jdbc {
        jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/cicada?characterEncoding=utf8"
        jdbc_user => "root"
        jdbc_password => "root123"
        jdbc_driver_library => "/usr/local/logstash/sync-config/mysql-connector-java-5.1.13.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_paging_enabled => "true"
        jdbc_page_size => "50000"
        jdbc_default_timezone => "Asia/Shanghai"
        statement_filepath => "/usr/local/logstash/sync-config/log_sql.sql"
        schedule => "* * * * *"
        type => "Log"
        lowercase_column_names => false
        record_last_run => true
        use_column_value => true
        tracking_column => "updateTime"
        tracking_column_type => "timestamp"
        last_run_metadata_path => "/usr/local/logstash/sync-config/log_last_time"
        clean_run => false
    }
}
filter {
    json {
        source => "message"
        remove_field => ["message"]
    }
}
output {
    if [type] == "User" {
        elasticsearch {
            hosts => ["127.0.0.1:9200"]
            index => "cicada_user_search"
            document_type => "user_search_index"
        }
    }
    if [type] == "Log" {
        elasticsearch {
            hosts => ["127.0.0.1:9200"]
            index => "cicada_log_search"
            document_type => "log_search_index"
        }
    }
}
```

2)、SQL文件

- user_sql.sql

```
SELECT
	id,
	user_name userName,
	user_phone userPhone,
	create_time createTime,
	update_time updateTime
FROM c_user
WHERE update_time > : sql_last_value
```
- log_sql.sql
```
SELECT
	id,
	param_value paramValue,
	request_ip requestIp,
	create_time createTime,
	update_time updateTime
FROM c_log
WHERE update_time > : sql_last_value
```

3)、配置参数说明

- input参数

```
statement_filepath：读取SQL语句位置
schedule ：这里配置每分钟执行一次
type ：类型，写入ES的标识
lowercase_column_names ：字段是否转小写
record_last_run ：记录上次执行时间
use_column_value ：使用列的值
tracking_column ：根据写入ES的updateTime字段区分增量数据
tracking_column_type ：区分的字段类型
```

- output参数

```
hosts ：ES服务地址
index ：Index名称，类比理解数据库名称
document_type ：Type名称，类比理解表名称
```

## 3、启动进程

```
/usr/local/logstash/bin/logstash  
-f  
/usr/local/logstash/sync-config/cicadaes.conf
```

# 二、ES客户端工具

**1、下载软件**

kibana-6.3.2-windows-x86_64

**2、修改配置**

kibana-6.3.2-windows-x86_64\config\kibana.yml

添加配置：

```
elasticsearch.url: "http://127.0.0.1:9200"
```

**3、双击启动**

kibana-6.3.2-windows-x86_64\bin\kibana.bat

**4、访问地址**

```
http://localhost:5601
```

![](https://images.gitee.com/uploads/images/2022/0214/232751_593e696b_5064118.jpeg "07-1.jpg")

**源码参考：** https://gitee.com/cicadasmile/linux-system-base