# 一、HDFS基本概述

## 1、HDFS描述

大数据领域一直面对的两大核心模块：数据存储，数据计算，HDFS作为最重要的大数据存储技术，具有高度的容错能力，稳定而且可靠。HDFS(Hadoop-Distributed-File-System)，它是一个分布式文件系统，用于存储文件，通过目录树来定位文件;设计初衷是管理数成百上千的服务器与磁盘，让应用程序像使用普通文件系统一样存储大规模的文件数据，适合一次写入，多次读出的场景，且不支持文件的修改，适合做数据分析。

## 2、基础架构

![](https://images.gitee.com/uploads/images/2022/0213/131627_a0e0ac52_5064118.png "04-1.png")

HDFS具有主/从体系结构，有两个核心组件，NameNode与DataNode。

**NameNode**

负责文件系统的元数据（MetaData）管理，即文件路径名、数据块ID、存储位置等信息，并配置副本策略，处理客户端读写请求。

**DataNode**

执行文件数据的实际存储和读写操作，每个DataNode存储一部分文件数据块，文件整体分布存储在整个HDFS服务器集群中。

**Client**

客户端，文件切分上传HDFS的时候，Client将文件切分成一个一个的Block，然后进行上传;从NameNode获取文件的位置信息;与DataNode通信读取或者写入数据; Client通过一些命令来访问或管理HDFS。

**Secondary-NameNode**

不是NameNode的热备，但是分担NameNode工作量，比如定期合并Fsimage和Edits，并推送给NameNode;在紧急情况下，可辅助恢复NameNode。

## 3、高容错性

![](https://images.gitee.com/uploads/images/2022/0213/131651_cec3b111_5064118.jpeg "04-2.jpg")

数据块多份复制存储的示意,文件/users/sameerp/data/part-0，复制备份设置为2，存储的block-ids分别为1、3；文件/users/sameerp/data/part-1，复制备份设置为3，存储的block-ids分别为2、4、5；任何单台服务器宕机后，每个数据块至少还存在一个备份服务存活，不会影响对文件的访问，提高整体容错性。

HDFS中的文件在物理上是分块存储(Block)，块的大小可以通过参数dfs.blocksize来配置，块设置太小，会增加寻址时间；块设置的太大，从磁盘传输数据的时间会很慢，HDFS块的大小设置主要取决于磁盘传输速率。

# 二、基础Shell命令

**1、基础命令**

查看Hadoop下相关Shell操作命令。

```
[root@hop01 hadoop2.7]# bin/hadoop fs
[root@hop01 hadoop2.7]# bin/hdfs dfs
```

dfs是fs的实现类

**2、查看命令描述**

```
[root@hop01 hadoop2.7]# hadoop fs -help ls
```

**3、递归创建目录**

```
[root@hop01 hadoop2.7]# hadoop fs -mkdir -p /hopdir/myfile
```

**4、查看目录**

```
[root@hop01 hadoop2.7]# hadoop fs -ls /
[root@hop01 hadoop2.7]# hadoop fs -ls /hopdir
```

**5、剪贴文件**

```
hadoop fs -moveFromLocal /opt/hopfile/java.txt /hopdir/myfile
## 查看文件
hadoop fs -ls /hopdir/myfile
```

**6、查看文件内容**

```
## 查看全部
hadoop fs -cat /hopdir/myfile/java.txt
## 查看末尾
hadoop fs -tail /hopdir/myfile/java.txt
```

**7、追加文件内容**

```
hadoop fs -appendToFile /opt/hopfile/c++.txt /hopdir/myfile/java.txt
```

**8、拷贝文件**

copyFromLocal命令和put命令相同

```
hadoop fs -copyFromLocal /opt/hopfile/c++.txt /hopdir
```

**9、HDFS文件拷贝到本地**

```
hadoop fs -copyToLocal /hopdir/myfile/java.txt /opt/hopfile/
```

**10、HDFS内拷贝文件**

```
hadoop fs -cp /hopdir/myfile/java.txt /hopdir
```

**11、HDFS内移动文件**

```
hadoop fs -mv /hopdir/c++.txt /hopdir/myfile
```

**12、合并下载多个文件**

基础命令get和copyToLocal命令效果相同。

```
hadoop fs -getmerge /hopdir/myfile/* /opt/merge.txt
```

**13、删除文件**

```
hadoop fs -rm /hopdir/myfile/java.txt
```

**14、查看文件夹信息**

```
hadoop fs -du -s -h /hopdir/myfile
```

**15、删除文件夹**

```
bin/hdfs dfs -rm -r /hopdir/file0703
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent/tree/master/series01-hadoop-parent