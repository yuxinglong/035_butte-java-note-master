# 一、Sqoop概述

Sqoop是一款开源的大数据组件，主要用来在Hadoop(Hive、HBase等)与传统的数据库(mysql、postgresql、oracle等)间进行数据的传递。

![](https://images.gitee.com/uploads/images/2022/0213/144147_1410c9f4_5064118.png "01-1.png")

通常数据搬运的组件基本功能：导入与导出。

鉴于Sqoop是大数据技术体系的组件，所以关系型数据库导入Hadoop存储系统称为导入，反过来称为导出。

Sqoop是一个命令行的组件工具，将导入或导出命令转换成mapreduce程序来实现。mapreduce中主要是对inputformat和outputformat进行定制。

# 二、环境部署

在测试Sqoop组件的时候，起码要具备Hadoop系列、关系型数据、JDK等基础环境。

鉴于Sqoop是工具类组件，单节点安装即可。

## 1、上传安装包

安装包和版本：`sqoop-1.4.6`

```
[root@hop01 opt]# tar -zxf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
[root@hop01 opt]# mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha sqoop1.4.6
```

## 2、修改配置文件

文件位置：`sqoop1.4.6/conf`

```
[root@hop01 conf]# pwd
/opt/sqoop1.4.6/conf
[root@hop01 conf]# mv sqoop-env-template.sh sqoop-env.sh
```

配置内容：涉及hadoop系列常用组件和调度组件zookeeper。

```
[root@hop01 conf]# vim sqoop-env.sh
# 配置内容
export HADOOP_COMMON_HOME=/opt/hadoop2.7
export HADOOP_MAPRED_HOME=/opt/hadoop2.7
export HIVE_HOME=/opt/hive1.2
export HBASE_HOME=/opt/hbase-1.3.1
export ZOOKEEPER_HOME=/opt/zookeeper3.4
export ZOOCFGDIR=/opt/zookeeper3.4
```

## 3、配置环境变量

```
[root@hop01 opt]# vim /etc/profile

export SQOOP_HOME=/opt/sqoop1.4.6
export PATH=$PATH:$SQOOP_HOME/bin

[root@hop01 opt]# source /etc/profile
```

## 4、引入MySQL驱动

```
[root@hop01 opt]# cp mysql-connector-java-5.1.27-bin.jar sqoop1.4.6/lib/
```

## 5、环境检查

![](https://images.gitee.com/uploads/images/2022/0213/144223_59e06736_5064118.png "01-2.png")

关键点：import与export

查看帮助命令，并通过version查看版本号。sqoop是一个基于命令行操作的工具，所以这里的命令下面还要使用。

## 6、相关环境

此时看下sqoop部署节点中的相关环境，基本都是集群模式：

![](https://images.gitee.com/uploads/images/2022/0213/144239_b5514add_5064118.png "01-3.png")

## 7、测试MySQL连接

```
sqoop list-databases --connect jdbc:mysql://hop01:3306/ --username root --password 123456
```

这里是查看MySQL数据库的命令，如图结果打印正确：

![](https://images.gitee.com/uploads/images/2022/0213/144307_6d2fe55b_5064118.png "01-4.png")

# 三、数据导入案例

## 1、MySQL数据脚本

```sql
CREATE TABLE `tb_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `user_name` varchar(100) DEFAULT NULL COMMENT '用户名',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
INSERT INTO `sq_import`.`tb_user`(`id`, `user_name`) VALUES (1, 'spring');
INSERT INTO `sq_import`.`tb_user`(`id`, `user_name`) VALUES (2, 'c++');
INSERT INTO `sq_import`.`tb_user`(`id`, `user_name`) VALUES (3, 'java');
```

![](https://images.gitee.com/uploads/images/2022/0213/144326_fdd73eeb_5064118.png "01-5.png")

## 2、Sqoop导入脚本

指定数据库的表，全量导入Hadoop系统，注意这里要启动Hadoop服务；

```
sqoop import 
--connect jdbc:mysql://hop01:3306/sq_import \
--username root \
--password 123456 \ 
--table tb_user \
--target-dir /hopdir/user/tbuser0 \
-m 1
```

## 3、Hadoop查询

![](https://images.gitee.com/uploads/images/2022/0213/144343_17a8c945_5064118.png "01-6.png")

```
[root@hop01 ~]# hadoop fs -cat /hopdir/user/tbuser0/part-m-00000
```

## 4、指定列和条件

查询的SQL语句中必须带有WHERE\$CONDITIONS：

```
sqoop import 
--connect jdbc:mysql://hop01:3306/sq_import \
--username root \
--password 123456 \
--target-dir /hopdir/user/tbname0 \
--num-mappers 1 \
--query 'select user_name from tb_user where 1=1 and $CONDITIONS;'
```

查看导出结果：

```
[root@hop01 ~]# hadoop fs -cat /hopdir/user/tbname0/part-m-00000
```

## 5、导入Hive组件

在不指定hive使用的数据库情况下，默认导入default库，并且自动创建表名称：

```
sqoop import 
--connect jdbc:mysql://hop01:3306/sq_import \
--username root \
--password 123456 \
--table tb_user \
--hive-import \
-m 1
```

执行过程，这里注意观察sqoop的执行日志即可：

第一步：MySQL的数据导入到HDFS的默认路径下；

第二步：把临时目录中的数据迁移到hive表中；

![](https://images.gitee.com/uploads/images/2022/0213/144415_ba9eac3f_5064118.png "01-7.png")

## 6、导入HBase组件

当前hbase的集群版本是1.3，需要先创建好表，才能正常执行数据导入：

```
sqoop import 
--connect jdbc:mysql://hop01:3306/sq_import \
--username root \
--password 123456 \
--table tb_user \
--columns "id,user_name" \
--column-family "info" \
--hbase-table tb_user \
--hbase-row-key id \
--split-by id
```

查看HBase中表数据：

![](https://images.gitee.com/uploads/images/2022/0213/144438_44fcbc46_5064118.png "01-8.png")

# 四、数据导出案例

新建一个MySQL数据库和表，然后把HDFS中的数据导出到MySQL中，这里就使用第一个导入脚本生成的数据即可：

![](https://images.gitee.com/uploads/images/2022/0213/144453_1a3774cd_5064118.png "01-9.png")

```
sqoop export 
--connect jdbc:mysql://hop01:3306/sq_export \
--username root \
--password 123456 \
--table tb_user \
--num-mappers 1 \
--export-dir /hopdir/user/tbuser0/part-m-00000 \
--num-mappers 1 \
--input-fields-terminated-by ","
```

再次查看MySQL中数据，记录完全被导出来，这里`,`是每个数据字段间的分隔符号，语法规则对照脚本一HDFS数据查询结果即可。

**源码参考：** https://gitee.com/cicadasmile/big-data-parent