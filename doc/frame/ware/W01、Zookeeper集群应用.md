

# 一、框架简介

## 1、基础简介

Zookeeper基于观察者模式设计的组件，主要应用于分布式系统架构中的，统一命名服务、统一配置管理、统一集群管理、服务器节点动态上下线、软负载均衡等场景。

## 2、集群选举

Zookeeper集群基于半数机制，集群中半数以上机器存活，集群处于可用状态。所以建议Zookeeper集群安装为奇数台服务器。在集群的配置文件中并没有指定Master和Slave。在Zookeeper工作时，是有一个节点为Leader，其他则为Follower，Leader是通过内部的选举机制临时产生的。

![](https://images.gitee.com/uploads/images/2022/0210/225850_d0592ad4_5064118.png "02-1.png")

**基本描述**

假设有三台服务器组成的Zookeeper集群，每个节点的myid编号依次1-3，依次启动服务器，会发现server2被选择为Leader节点。

server1启动，执行一次选举。服务器1投自己一票。此时服务器1票数一票，未达到半数以上（2票），选举无法完成，服务器1状态保持为LOOKING；

server2启动，再执行一次选举。服务器1和2分别投自己一票，并交换选票信息，因为服务器2的myid比服务器1的myid大，服务器1会更改选票为投服务器2。此时服务器1票数0票，服务器2票数2票，达到半数以上，选举完成，服务器1状态为follower，2状态保持leader，此时集群可用，服务器3启动后直接为follower。

# 二、集群配置

## 1、创建配置目录

```
# mkdir -p /data/zookeeper/data
# mkdir -p /data/zookeeper/logs
```

## 2、基础配置

```
# vim /opt/zookeeper-3.4.14/conf/zoo.cfg

tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
```

## 3、单节点配置

```
# vim /data/zookeeper/data/myid 
```

三个节点服务，分别在myid文件中写入[1,2,3]

## 4、集群服务

在每个服务的zoo.cfg配置文件中写入如下配置：

```
server.1=192.168.72.133:2888:3888
server.2=192.168.72.136:2888:3888
server.3=192.168.72.137:2888:3888
```

## 5、启动集群

分别启动三台zookeeper服务

```
[zookeeper-3.4.14]# bin/zkServer.sh start
Starting zookeeper ... STARTED
```

## 6、查看集群状态

Mode: leader是Master节点

Mode: follower是Slave节点

```
[zookeeper-3.4.14]# bin/zkServer.sh status
Mode: leader
```

## 7、集群状态测试

随便登录一台服务的客户端，创建一个测试节点，然后在其他服务上查看。

```
[zookeeper-3.4.14 bin]# ./zkCli.sh
[zk: 0] create /node-test01 node-test01  
Created /node-test01
[zk: 1] get /node-test01
```

或者关闭leader节点

```
[zookeeper-3.4.14 bin]# ./zkServer.sh stop
```

则会重新选举该节点。

## 8、Nginx统一管理

```
[rnginx-1.15.2 conf]# vim nginx.conf

stream {
    upstream zkcluster {
        server 192.168.72.133:2181;
        server 192.168.72.136:2181;
        server 192.168.72.136:2181;
    }
    server {
        listen 2181;
        proxy_pass zkcluster;
    }
}
```

# 三、服务节点监听

## 1、基本原理

分布式系统中，主节点可以有多台，可以动态上下线，任意一台客户端都能实时感知到主节点服务器的上下线。

![](https://images.gitee.com/uploads/images/2022/0210/225905_e720917d_5064118.png "02-2.png")

流程描述：

- 启动Zookeeper集群服务；
- RegisterServer模拟服务端注册；
- ClientServer模拟客户端监听；
- 启动服务端注册三次，注册不同节点的zk-node服务；
- 依次关闭注册的服务端，模拟服务下线流程；
- 查看客户端日志，可以监控到服务节点变化；

首先创建一个节点：serverList，用来存放服务器列表。

```
[zk: 0] create /serverList "serverList" 
```

## 2、服务端注册

```java
package com.zkper.cluster.monitor;
import java.io.IOException;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.ZooDefs.Ids;

public class RegisterServer {

    private ZooKeeper zk ;
    private static final String connectString = "127.0.0.133:2181,127.0.0.136:2181,127.0.0.137:2181";
    private static final int sessionTimeout = 3000;
    private static final String parentNode = "/serverList";

    private void getConnect() throws IOException{
        zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
            }
        });
    }

    private void registerServer(String nodeName) throws Exception{
        String create = zk.create(parentNode + "/server", nodeName.getBytes(),
                                  Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println(nodeName +" 上线："+ create);
    }

    private void working() throws Exception{

        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws Exception {
        RegisterServer server = new RegisterServer();
        server.getConnect();
        // 分别启动三次服务，注册不同节点，再一次关闭不同服务端看客户端效果
        // server.registerServer("zk-node-133");
        // server.registerServer("zk-node-136");
        server.registerServer("zk-node-137");
        server.working();
    }
}
```

## 3、客户端监听

```java
package com.zkper.cluster.monitor;
import org.apache.zookeeper.*;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class ClientServer {
    private ZooKeeper zk ;
    private static final String connectString = "127.0.0.133:2181,127.0.0.136:2181,127.0.0.137:2181";
    private static final int sessionTimeout = 3000;
    private static final String parentNode = "/serverList";

    private void getConnect() throws IOException {
        zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                try {
                    // 监听在线的服务列表
                    getServerList();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }

    private void getServerList() throws Exception {
        List<String> children = zk.getChildren(parentNode, true);
        List<String> servers = new ArrayList<>();
        for (String child : children) {
            byte[] data = zk.getData(parentNode + "/" + child, false, null);
            servers.add(new String(data));
        }
        System.out.println("当前服务列表："+servers);
    }

    private void working() throws Exception{
        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws Exception {
        ClientServer client = new ClientServer();
        client.getConnect();
        client.getServerList();
        client.working();
    }

}
```

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap03-frame-design/case02-zkper-cluster