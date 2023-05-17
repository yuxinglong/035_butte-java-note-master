# 一、Hive基础简介

**1、基础描述**

Hive是基于Hadoop的一个数据仓库工具，用来进行数据提取、转化、加载，是一个可以对Hadoop中的大规模存储的数据进行查询和分析存储的组件，Hive数据仓库工具能将结构化的数据文件映射为一张数据库表，并提供SQL查询功能，能将SQL语句转变成MapReduce任务来执行，使用成本低，可以通过类似SQL语句实现快速MapReduce统计，使MapReduce变得更加简单，而不必开发专门的MapReduce应用程序。hive十分适合对数据仓库进行统计分析。

**2、组成与架构**

![](https://images.gitee.com/uploads/images/2022/0213/141247_0c51d383_5064118.png "01-1.png")

**用户接口**：ClientCLI、JDBC访问Hive、WEBUI浏览器访问Hive。

**元数据**：Hive将元数据存储在数据库中，如mysql、derby。Hive中的元数据包括表的名字，表的列和分区以及属性，表的属性（是否为外部表等），表的数据所在目录等。

**驱动器**：基于解释器、编辑器、优化器完成HQL查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。

**执行器引擎**：ExecutionEngine把逻辑执行计划转换成可以运行的物理计划。

**Hadoop底层**：基于HDFS进行存储，使用MapReduce进行计算，基于Yarn的调度机制。

Hive收到给客户端发送的交互请求，接收到操作指令(SQL)，并将指令翻译成MapReduce，提交到Hadoop中执行，最后将执行结果输出到客户端。

# 二、Hive环境安装

**1、准备安装包**

hive-1.2，依赖Hadoop集群环境，位置放在hop01服务上。

**2、解压重命名**

```
tar -zxvf apache-hive-1.2.1-bin.tar.gz
mv apache-hive-1.2.1-bin/ hive1.2
```

**3、修改配置文件**

创建配置文件

```
[root@hop01 conf]# pwd
/opt/hive1.2/conf
[root@hop01 conf]# mv hive-env.sh.template hive-env.sh
```

添加内容

```
[root@hop01 conf]# vim hive-env.sh
export HADOOP_HOME=/opt/hadoop2.7
export HIVE_CONF_DIR=/opt/hive1.2/conf
```

配置内容一个是Hadoop路径，和hive配置文件路径。

**4、Hadoop配置**

首先启动hdfs和yarn；然后在HDFS上创建/tmp和/user/hive/warehouse两个目录并修改赋予权限。

```
bin/hadoop fs -mkdir /tmp
bin/hadoop fs -mkdir -p /user/hive/warehouse
bin/hadoop fs -chmod g+w /tmp
bin/hadoop fs -chmod g+w /user/hive/warehouse
```

**5、启动Hive**

```
[root@hop01 hive1.2]# bin/hive
```

**6、基础操作**

**查看数据库**

```
hive> show databases ;
```

**选择数据库**

```
hive> use default;
```

**查看数据表**

```
hive> show tables;
```

**创建数据库使用**

```
hive> create database mytestdb;
hive> show databases ;
default
mytestdb
hive> use mytestdb;
```

**创建表**

```
create table hv_user (id int, name string, age int);
```

**查看表结构**

```
hive> desc hv_user;
id                  	int                 	                    
name                	string              	                    
age                 	int 
```

**添加表数据**

```
insert into hv_user values (1, "test-user", 23);
```

**查询表数据**

```
hive> select * from hv_user ;
```

注意：这里通过对查询日志的观察，明显看出Hive执行的流程。

**删除表**

```
hive> drop table hv_user ;
```

**退出Hive**

```
hive> quit;
```

**查看Hadoop目录**

```
# hadoop fs -ls /user/hive/warehouse       
/user/hive/warehouse/mytestdb.db
```

通过Hive创建的数据库和数据存储在HDFS上。

# 三、整合MySQL5.7环境

这里默认安装好MySQL5.7的版本，并配置好相关登录账号，配置root用户的Host为%模式。

**1、上传MySQL驱动包**

将MySQL驱动依赖包上传到hive安装目录的lib目录下。

```
[root@hop01 lib]# pwd
/opt/hive1.2/lib
[root@hop01 lib]# ll
mysql-connector-java-5.1.27-bin.jar
```

**2、创建hive-site配置**

```
[root@hop01 conf]# pwd
/opt/hive1.2/conf
[root@hop01 conf]# touch hive-site.xml
[root@hop01 conf]# vim hive-site.xml
```

**3、配置MySQL存储**

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
        <property>
          <name>javax.jdo.option.ConnectionURL</name>
          <value>jdbc:mysql://hop01:3306/metastore?createDatabaseIfNotExist=true</value>
          <description>JDBC connect string for a JDBC metastore</description>
        </property>

        <property>
          <name>javax.jdo.option.ConnectionDriverName</name>
          <value>com.mysql.jdbc.Driver</value>
          <description>Driver class name for a JDBC metastore</description>
        </property>

        <property>
          <name>javax.jdo.option.ConnectionUserName</name>
          <value>root</value>
          <description>username to use against metastore database</description>
        </property>

        <property>
          <name>javax.jdo.option.ConnectionPassword</name>
          <value>123456</value>
          <description>password to use against metastore database</description>
        </property>
</configuration>
```

配置完成后，依次重启MySQL、hadoop、hive环境，查看MySQL数据库信息，多了metastore数据库和相关表。

**4、后台启动hiveserver2**

```
[root@hop01 hive1.2]# bin/hiveserver2 &
```

**5、Jdbc连接测试**

```
[root@hop01 hive1.2]# bin/beeline
Beeline version 1.2.1 by Apache Hive
beeline> !connect jdbc:hive2://hop01:10000
Connecting to jdbc:hive2://hop01:10000
Enter username for jdbc:hive2://hop01:10000: hiveroot (账户回车)
Enter password for jdbc:hive2://hop01:10000: ******   (密码123456回车)
Connected to: Apache Hive (version 1.2.1)
Driver: Hive JDBC (version 1.2.1)
0: jdbc:hive2://hop01:10000> show databases;
+----------------+--+
| database_name  |
+----------------+--+
| default        |
+----------------+--+
```

# 四、高级查询语法

**1、基础函数**

```sql
select count(*) count_user from hv_user;
select sum(age) sum_age from hv_user;
select min(age) min_age,max(age) max_age from hv_user;
+----------+----------+--+
| min_age  | max_age  |
+----------+----------+--+
| 23       | 25       |
+----------+----------+--+
```

**2、条件查询语句**

```sql
select * from hv_user where name='test-user' limit 1;
+-------------+---------------+--------------+--+
| hv_user.id  | hv_user.name  | hv_user.age  |
+-------------+---------------+--------------+--+
| 1           | test-user     | 23           |
+-------------+---------------+--------------+--+

select * from hv_user where id>1 AND name like 'dev%';
+-------------+---------------+--------------+--+
| hv_user.id  | hv_user.name  | hv_user.age  |
+-------------+---------------+--------------+--+
| 2           | dev-user      | 25           |
+-------------+---------------+--------------+--+

select count(*) count_name,name from hv_user group by name;
+-------------+------------+--+
| count_name  |    name    |
+-------------+------------+--+
| 1           | dev-user   |
| 1           | test-user  |
+-------------+------------+--+
```

**3、连接查询**

```sql
select t1.*,t2.* from hv_user t1 join hv_dept t2 on t1.id=t2.dp_id;
+--------+------------+---------+-----------+-------------+--+
| t1.id  |  t1.name   | t1.age  | t2.dp_id  | t2.dp_name  |
+--------+------------+---------+-----------+-------------+--+
| 1      | test-user  | 23      | 1         | 技术部      |
+--------+------------+---------+-----------+-------------+--+
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent