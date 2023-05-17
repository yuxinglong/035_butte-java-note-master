# 一、下载解压

## 1、Zookeeper简介

Zookeeper 作为一个分布式的服务框架，主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储，但是 Zookeeper 并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化。通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。

## 2、下载

环境版本

- centos7
- zookeeper 3.4.14

```
[root@localhost mysoft]$ cd /usr/local/mysoft/
[root@localhost mysoft]$
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
[root@localhost mysoft]# tar -zxvf zookeeper-3.4.14.tar.gz
[root@localhost mysoft]# mv zookeeper-3.4.14 zookeeper3.4
```

# 二、修改配置文件

## 1、数据和日志目录

```
[root@localhost /]# mkdir -p data/log/zkp1.log
[root@localhost /]# mkdir -p data/zkpdata/zkp1
```

## 2、修改配置

```
[root@localhost mysoft]# cd zookeeper3.4/conf/
[root@localhost conf]# cp zoo_sample.cfg zoo.cfg
[root@localhost conf]# vim zoo.cfg
# 修改如下两块内容，其他默认
dataDir=/data/zkpdata/zkp1
dataLogDir=/data/log/zkp1.log
```

## 3、配置文件说明

```
1）tickTime
心跳检查的时间。
2）initLimit
集群中的从服务器与主服务器之间初始连接时能容忍的最多心跳数（tickTime的数量）。
3）syncLimit
集群中从服务器与主服务器之间的请求和答应最多能容忍的心跳数。
4）dataDir
数据存放目录。
5）dataLogDir
日志存放目录。
6）clientPort
客户端连接的接口，客户端连接zookeeper服务器的端口，服务器端会监听这个端口，默认是2181。
```

# 三、启动运行

## 1、启动服务端

```
[root@localhost bin]# pwd
/usr/local/mysoft/zookeeper3.4/bin
[root@localhost bin]# /usr/local/mysoft/zookeeper3.4/bin/zkServer.sh start /usr/local/mysoft/zookeeper3.4/conf/zoo.cfg 

ZooKeeper JMX enabled by default
Using config: /usr/local/mysoft/zookeeper3.4/conf/zoo.cfg
Starting zookeeper ... STARTED
[root@localhost bin]# ps -aux |grep zookeeper
```

## 2、启动客户端

```
[root@localhost /]# cd /usr/local/mysoft/zookeeper3.4/bin/
[root@localhost bin]# ./zkCli.sh 
Connecting to localhost:2181
```

# 四、常用操作命令

```
## 创建节点
[zk: localhost:2181(CONNECTED) 2] create /cicada cicada-smile1
Created /cicada
[zk: localhost:2181(CONNECTED) 8] create /cicada2 cicada-smile2
Created /cicada2
[zk: localhost:2181(CONNECTED) 4] get /cicada
cicada-smile1
## 查看目录 
[zk: localhost:2181(CONNECTED) 5] ls /
[zookeeper, cicada, cicada2] 
## 查看指定目录
[zk: localhost:2181(CONNECTED) 17] ls / zookeeper
[com.ptp.user.service.UserService]
## 删除节点
[zk: localhost:2181(CONNECTED) 10] delete /cicada
## 删除目录全部
[zk: localhost:2181(CONNECTED) 18] rmr /cicada2
[zk: localhost:2181(CONNECTED) 19] ls /cicada2
Node does not exist: /cicada2
## 查看剩下节点
[zk: localhost:2181(CONNECTED) 13] ls /
[zookeeper]
```

**源码参考：** https://gitee.com/cicadasmile/linux-system-base