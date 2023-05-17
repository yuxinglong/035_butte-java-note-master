# 一、Azkaban概述

## 1、任务时序

在数据服务的业务场景中，很常见的业务流程就是日志文件经过大数据分析，再向业务输出结果数据；在该过程中会有很多任务需要执行，并且很难精准把握任务执行的结束时间，但是又希望整个任务链尽快结束释放资源。

![](https://images.gitee.com/uploads/images/2022/0213/145027_564e68cd_5064118.png "01-1.png")

大致执行顺序如下：

- 业务日志文件同步到HDFS文件系统；
- 经过Hadoop执行分析计算过程；
- 结果数据在导入数仓进行存储；
- 最终需要把数仓内数据同步到业务库；

这样的流程不必业务中任务调度，时间基本是可预估的，只要把握留足任务间隔时间即可，大数据的任务链路通常需要一个结束直接启动另一个，以此降低时间成本，初入数据服务公司时，就发生过因为同步任务执行结束但是最后的个别CSV数据文件未生成结束的案例，导致近百万的分析数据同步更新业务库失败。

## 2、Azkaban简介

Azkaban是由Linkedin公司推出的可以管理批量工作流任务的调度器，用于在一个工作流内以一个特定的顺序运行一组工作和流程。Azkaban使用job配置文件建立任务之间的依赖关系，并提供一个易于使用的web用户界面维护和跟踪你的工作流。

Azkaban特点和优势

- 提供功能清晰，简单易用的 Web UI 界面；
- 作业配置简单，任务作业依赖关系清晰；
- 提供可扩展的组件；
- 基于Java语言开发，易于二次开发；

相比较于Oozie配置工作流的过程是编写大量的XML配置，并且其代码复杂度比较高，不易于二次开发，Azkaban则显得轻量级，功能和用法相对简单和容易使用。

# 二、服务安装

## 1、核心包

**Web服务**

```
azkaban-web-server-2.5.0.tar.gz
```

**执行服务**

```
azkaban-executor-server-2.5.0.tar.gz
```

**SQL脚本**

```
azkaban-sql-script-2.5.0.tar.gz
```

## 2、安装路径

上传上面三个安装包，并解压操作。

```
[root@hop01 azkaban]# pwd
/opt/azkaban
[root@hop01 azkaban]# tar -zxvf azkaban-web-server-2.5.0.tar.gz
[root@hop01 azkaban]# tar -zxvf azkaban-executor-server-2.5.0.tar.gz
[root@hop01 azkaban]# tar -zxvf azkaban-sql-script-2.5.0.tar.gz
[root@hop01 azkaban]# mv azkaban-web-2.5.0/ server
[root@hop01 azkaban]# mv azkaban-executor-2.5.0/ executor
```

## 3、MySQL导入脚本

```sql
[root@hop01 ~]# mysql -uroot -p123456
mysql> create database azkaban_test;
mysql> use azkaban_test;
mysql> source /opt/azkaban/azkaban-2.5.0/create-all-sql-2.5.0.sql
```

查看表

![](https://images.gitee.com/uploads/images/2022/0213/145047_f2b40ef0_5064118.png "01-2.png")

## 4、SSL配置

```
[root@hop01 opt]# keytool -keystore keystore -alias jetty -genkey -keyalg RSA
```

生成文件：`keystore`

拷贝到AzkabanWeb服务器目录下：

```
[root@hop01 opt]# mv keystore /opt/azkaban/server/
```

## 5、Web服务配置

**基础配置**

```
[root@hop01 conf]# pwd
/opt/azkaban/server/conf
[root@hop01 conf]# vim azkaban.properties
```

核心修改内容：MySQL和Jetty。

```
default.timezone.id=Asia/Shanghai

# Azkaban MySQL server properties.
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban_test
mysql.user=root
mysql.password=123456
mysql.numconnections=100

# Azkaban Jetty server properties.
jetty.maxThreads=25
jetty.ssl.port=8443
jetty.port=8081
jetty.keystore=keystore
jetty.password=123456
jetty.keypassword=123456
jetty.truststore=keystore
jetty.trustpassword=123456
```

这里配置符合本地配置参数即可。

**用户配置**

```
[root@hop01 conf]# vim azkaban-users.xml
```

增加一个管理员用户：

```xml
<azkaban-users>
    <user username="admin" password="admin" roles="admin,metrics" />
</azkaban-users>
```

![](https://images.gitee.com/uploads/images/2022/0213/145114_b8d833a4_5064118.png "01-3.png")

## 6、Executor服务配置

```
[root@hop01 conf]# pwd
/opt/azkaban/executor/conf
[root@hop01 conf]# vim azkaban.properties
```

核心修改内容：MySQL和时区。

```
default.timezone.id=Asia/Shanghai

# Azkaban MySQL server properties.
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban_test
mysql.user=root
mysql.password=123456
mysql.numconnections=100
```

## 7、启动服务器

**Web服务**

```
[root@hop01 bin]# pwd
/opt/azkaban/server/bin
[root@hop01 bin]# ll
total 16
-rwxr-xr-x 1 root root  161 Apr 21  2014 azkaban-web-shutdown.sh
-rwxr-xr-x 1 root root 1275 Apr 21  2014 azkaban-web-start.sh
```

这里分别是启动和关闭的脚本。

```
[root@hop01 bin]# /opt/azkaban/server/bin/azkaban-web-start.sh
```

**Executor服务**

```
[root@hop01 bin]# /opt/azkaban/executor/bin/azkaban-executor-start.sh
```

**启动日志**

两个服务的关键尾行日志：

```
Azkaban Server running on ssl port 8443.
Azkaban Executor Server started on port 12321
```

**登录界面**

注意这里是基于https协议：

```
https://hop01:8443/
```

![](https://images.gitee.com/uploads/images/2022/0213/145131_2284c1dc_5064118.png "01-4.png")

# 三、操作案例

## 1、入门案例

**创建command类型job**

```
[root@hop01 flow_01]# pwd
/opt/azkaban/testJob/flow_01
[root@hop01 flow_01]# vim simple.job

type=command
command=echo 'mySimpleJob'
```

**打成zip包**

```
[root@hop01 flow_01]# zip -q -r simpleJob.zip simple.job
```

**创建项目**

![](https://images.gitee.com/uploads/images/2022/0213/151233_ee3c4c96_5064118.png "01-5.png")

**上传任务包**

![](https://images.gitee.com/uploads/images/2022/0213/151244_c3ed7f4c_5064118.png "01-6.png")

**执行任务**

![](https://images.gitee.com/uploads/images/2022/0213/151255_1081e99c_5064118.png "01-7.png")

## 2、任务顺序执行

**创建任务A**

```
[root@hop01 flow_02]# vim simpleA.job

type=command
command=echo 'simplejobA'
```

**创建任务B**

```
[root@hop01 flow_02]# vim simpleB.job

type=command
dependencies=simpleA
command=echo 'simplejobB'
```

**打包任务**

```
[root@hop01 flow_02]# zip -q -r simpleTwoJob.zip simpleA.job simpleB.job
```

![](https://images.gitee.com/uploads/images/2022/0213/151307_3ca7cee8_5064118.png "01-8.png")

同样的操作方式，两个任务放在zip包中，通过Web服务上传，观察执行效果即可。

**源码参考：** https://gitee.com/cicadasmile/big-data-parent