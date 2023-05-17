# 一、HDFS高可用

## 1、基础描述

在单点或者少数节点故障的情况下，集群还可以正常的提供服务，HDFS高可用机制可以通过配置Active/Standby两个NameNodes节点实现在集群中对NameNode的热备来消除单节点故障问题，如果单个节点出现故障，可通过该方式将NameNode快速切换到另外一个节点上。

## 2、机制详解

![](https://images.gitee.com/uploads/images/2022/0213/133056_7bafe397_5064118.png "08-1.png")

- 基于两个NameNode做高可用，依赖共享Edits文件和Zookeeper集群；
- 每个NameNode节点配置一个ZKfailover进程，负责监控所在NameNode节点状态；
- NameNode与ZooKeeper集群维护一个持久会话；
- 如果Active节点故障停机，ZooKeeper通知Standby状态的NameNode节点；
- 在ZKfailover进程检测并确认故障节点无法工作后；
- ZKfailover通知Standby状态的NameNode节点切换为Active状态继续服务；

ZooKeeper在大数据体系中非常重要，协调不同组件的工作，维护并传递数据，例如上述高可用下自动故障转移就依赖于ZooKeeper组件。

# 二、HDFS高可用

## 1、整体配置

|服务列表 | HDFS文件 | YARN调度 | 单服务 | 共享文件 | Zk集群 |
|---|---|---|---|---|---|
|hop01 | DataNode | NodeManager | NameNode | JournalNode | ZK-hop01 |
|hop02 | DataNode | NodeManager | ResourceManager | JournalNode | ZK-hop02 |
|hop03 | DataNode | NodeManager | SecondaryNameNode | JournalNode | ZK-hop03 |

## 2、配置JournalNode

**创建目录**

```
[root@hop01 opt]# mkdir hopHA
```

**拷贝Hadoop目录**

```
cp -r /opt/hadoop2.7/ /opt/hopHA/
```

**配置core-site.xml**

```xml
<configuration>
    <!-- NameNode集群模式 -->
	<property>
		<name>fs.defaultFS</name>
       	<value>hdfs://mycluster</value>
	</property>
	<!-- 指定hadoop运行时产生文件的存储目录 -->
	<property>
		<name>hadoop.tmp.dir</name>
	   <value>/opt/hopHA/hadoop2.7/data/tmp</value>
	</property>
</configuration>
```

**配置hdfs-site.xml**，添加内容如下

```xml
<!-- 分布式集群名称 -->
<property>
	<name>dfs.nameservices</name>
	<value>mycluster</value>
</property>

<!-- 集群中NameNode节点 -->
<property>
	<name>dfs.ha.namenodes.mycluster</name>
	<value>nn1,nn2</value>
</property>

<!-- NN1 RPC通信地址 -->
<property>
	<name>dfs.namenode.rpc-address.mycluster.nn1</name>
	<value>hop01:9000</value>
</property>

<!-- NN2 RPC通信地址 -->
<property>
	<name>dfs.namenode.rpc-address.mycluster.nn2</name>
	<value>hop02:9000</value>
</property>

<!-- NN1 Http通信地址 -->
<property>
	<name>dfs.namenode.http-address.mycluster.nn1</name>
	<value>hop01:50070</value>
</property>

<!-- NN2 Http通信地址 -->
<property>
	<name>dfs.namenode.http-address.mycluster.nn2</name>
	<value>hop02:50070</value>
</property>

<!-- 指定NameNode元数据在JournalNode上的存放位置 -->
<property>
	<name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hop01:8485;hop02:8485;hop03:8485/mycluster</value>
</property>

<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
<property>
	<name>dfs.ha.fencing.methods</name>
	<value>sshfence</value>
</property>

<!-- 使用隔离机制时需要ssh无秘钥登录-->
<property>
	<name>dfs.ha.fencing.ssh.private-key-files</name>
	<value>/root/.ssh/id_rsa</value>
</property>

<!-- 声明journalnode服务器存储目录-->
<property>
	<name>dfs.journalnode.edits.dir</name>
	<value>/opt/hopHA/hadoop2.7/data/jn</value>
</property>

<!-- 关闭权限检查-->
<property>
	<name>dfs.permissions.enable</name>
	<value>false</value>
</property>

<!-- 访问代理类失败自动切换实现方式-->
<property>
	<name>dfs.client.failover.proxy.provider.mycluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
```

**依次启动journalnode服务**

```
[root@hop01 hadoop2.7]# pwd
/opt/hopHA/hadoop2.7
[root@hop01 hadoop2.7]# sbin/hadoop-daemon.sh start journalnode
```

**删除hopHA下数据**

```
[root@hop01 hadoop2.7]# rm -rf data/ logs/
```

**NN1格式化并启动NameNode**

```
[root@hop01 hadoop2.7]# pwd
/opt/hopHA/hadoop2.7
bin/hdfs namenode -format
sbin/hadoop-daemon.sh start namenode
```

**NN2同步NN1数据**

```
[root@hop02 hadoop2.7]# bin/hdfs namenode -bootstrapStandby
```

**NN2启动NameNode**

```
[root@hop02 hadoop2.7]# sbin/hadoop-daemon.sh start namenode
```

**查看当前状态**

![](https://images.gitee.com/uploads/images/2022/0213/133128_312e6355_5064118.png "08-2.png")

**在NN1上启动全部DataNode**

```
[root@hop01 hadoop2.7]# sbin/hadoop-daemons.sh start datanode
```

**NN1切换为Active状态**

```
[root@hop01 hadoop2.7]# bin/hdfs haadmin -transitionToActive nn1
[root@hop01 hadoop2.7]# bin/hdfs haadmin -getServiceState nn1
active
```

![](https://images.gitee.com/uploads/images/2022/0213/133140_9084e012_5064118.png "08-3.png")

## 3、故障转移配置

*配置hdfs-site.xml*，新增内容如下，同步集群

```xml
<property>
	<name>dfs.ha.automatic-failover.enabled</name>
	<value>true</value>
</property>
```

**配置core-site.xml**，新增内容如下，同步集群

```xml
<property>
	<name>ha.zookeeper.quorum</name>
	<value>hop01:2181,hop02:2181,hop03:2181</value>
</property>
```

**关闭全部HDFS服务**

```
[root@hop01 hadoop2.7]# sbin/stop-dfs.sh
```

**启动Zookeeper集群**

```
/opt/zookeeper3.4/bin/zkServer.sh start
```

**hop01初始化HA在Zookeeper中状态**

```
[root@hop01 hadoop2.7]# bin/hdfs zkfc -formatZK
```

**hop01启动HDFS服务**

```
[root@hop01 hadoop2.7]# sbin/start-dfs.sh
```

**NameNode节点启动ZKFailover**

这里hop01和hop02先启动的服务状态就是Active，这里先启动hop02。

```
[hadoop2.7]# sbin/hadoop-daemon.sh start zkfc
```

![](https://images.gitee.com/uploads/images/2022/0213/133200_eaec3e10_5064118.png "08-4.png")

**结束hop02的NameNode进程**

```
kill -9 14422
```

**等待一下查看hop01状态**

```
[root@hop01 hadoop2.7]# bin/hdfs haadmin -getServiceState nn1
active
```

# 三、YARN高可用

## 1、基础描述

![](https://images.gitee.com/uploads/images/2022/0213/135836_3a5163a4_5064118.png "08-5.png")

基本流程和思路与HDFS机制类似，依赖Zookeeper集群，当Active节点故障时，Standby节点会切换为Active状态持续服务。

## 2、配置详解

环境同样基于hop01和hop02来演示。

**配置yarn-site.xml**，同步集群下服务

```xml
<configuration>

    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <!--启用HA机制-->
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
 
    <!--声明Resourcemanager服务-->
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>cluster-yarn01</value>
    </property>

    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hop01</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>hop02</value>
    </property>
 
    <!--Zookeeper集群的地址--> 
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>hop01:2181,hop02:2181,hop03:2181</value>
    </property>

    <!--启用自动恢复机制--> 
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
 
    <!--指定状态存储Zookeeper集群--> 
    <property>
        <name>yarn.resourcemanager.store.class</name>     <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>

</configuration>
```

**重启journalnode节点**

```
sbin/hadoop-daemon.sh start journalnode
```

**在NN1服务格式化并启动**

```
[root@hop01 hadoop2.7]# bin/hdfs namenode -format
[root@hop01 hadoop2.7]# sbin/hadoop-daemon.sh start namenode
```

**NN2上同步NN1元数据**

```
[root@hop02 hadoop2.7]# bin/hdfs namenode -bootstrapStandby
```

**启动集群下DataNode**

```
[root@hop01 hadoop2.7]# sbin/hadoop-daemons.sh start datanode
```

**NN1设置为Active状态**

先启动hop01即可，然后启动hop02。

```
[root@hop01 hadoop2.7]# sbin/hadoop-daemon.sh start zkfc
```

**hop01启动yarn**

```
[root@hop01 hadoop2.7]# sbin/start-yarn.sh
```

**hop02启动ResourceManager**

```
[root@hop02 hadoop2.7]# sbin/yarn-daemon.sh start resourcemanager
```

**查看状态**

```
[root@hop01 hadoop2.7]# bin/yarn rmadmin -getServiceState rm1
```

![](https://images.gitee.com/uploads/images/2022/0213/135858_e0358837_5064118.png "08-6.png")

**源码参考：** https://gitee.com/cicadasmile/big-data-parent/tree/master/series01-hadoop-parent