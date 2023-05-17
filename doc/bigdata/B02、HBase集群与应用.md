# 一、Hbase简介

**1、基础描述**

Hadoop原生的特点是解决大规模数据的离线批量处理场景，HDFS具备强大存储能力，但是并没有提供很强的数据查询机制。HBase组件则是基于HDFS文件系统之上提供类似于BigTable服务。

HBase是一种分布式、可扩展、支持海量结构化数据存储的NoSQL数据库。HBase在Hadoop之上提供了类似于Bigtable的能力，基于列存储模式的而不是基于行的模式。存储数据特点：非结构化或者松散的半结构化数据，存储大表自然是需要具备水平扩展的能力，基于服务集群处理海量庞大数据。

**2、数据模型**

基于Hbase的数据结构的基本描述；

- 表-Table：由行和列组成，列划分为若干个列族；
- 行-Row：行键(Key)作标识，行代表数据对象；
- 列族：列族支持动态扩展，以字符串形式存储；
- 列标识：列族中的数据通过列标识符来定位；
- 单元格：行键，列族，列标识符共同确定一个单元；
- 单元数据：存储在单元里的数据称为单元数据；
- 时间戳：默认基于时间戳来进行版本标识；

HBase的数据模型同关系型数据库很类似，数据存储在一张表中，有行有列。但从HBase的底层物理存储结构看更像是Map（K-V）集合。

- 数据管理是基于列存储的特点；
- 简单的数据模型，内容存储为字符串；
- 没有复杂的表关系，简单的增删查操作；

从整体上看数据模型，HBase是一个稀疏、多维度、排序的映射表，这张表的索引是行键、列族、列限定符和时间戳每个值是一个未经解释的字符串。

# 二、搭建集群环境

**1、解压文件**

```
tar -zxvf hbase-1.3.1-bin.tar.gz
```

**2、配置环境变量**

```
vim /etc/profile

export HBASE_HOME=/opt/hbase-1.3.1
export PATH=$PATH:$HBASE_HOME/bin

source /etc/profile
```

**3、配置：hbase-env**

```
vim /opt/hbase-1.3.1/conf/hbase-env.sh

export JAVA_HOME=/opt/jdk1.8
export HBASE_MANAGES_ZK=false
```

**4、配置：hbase-site**

```xml
vim /opt/hbase-1.3.1/conf/hbase-site.xml

<configuration>
    <!--HDFS存储-->
	<property>
		<name>hbase.rootdir</name>
		<value>hdfs://hop01:9000/HBase</value>
	</property>
    <!--开启集群-->
	<property>
		<name>hbase.cluster.distributed</name>
		<value>true</value>
	</property>
	<!-- 端口 -->
	<property>
		<name>hbase.master.port</name>
		<value>16000</value>
	</property>
    <!--ZK集群-->
	<property>   
		<name>hbase.zookeeper.quorum</name>
	     <value>hop01,hop02,hop03</value>
	</property>
    <!--ZK数据--> 
	<property>   
		<name>hbase.zookeeper.property.dataDir</name>
	     <value>/data/zookeeper/data/</value>
	</property>
</configuration>
```

**5、配置：regionservers**

```
vim /opt/hbase-1.3.1/conf/regionservers

hop01
hop02
hop03
```

**6、配置：软连接**

软连接hadoop配置文件到HBase

```
ln -s /opt/hadoop2.7/etc/hadoop/core-site.xml /opt/hbase-1.3.1/conf/core-site.xml
ln -s /opt/hadoop2.7/etc/hadoop/hdfs-site.xml /opt/hbase-1.3.1/conf/hdfs-site.xml
```

**7、同步集群服务环境**

也可以手动配置集群，或者使用同步命令。

```
xsync hbase/
```

**8、启动集群**

在hop01节点启动即可。

```
/opt/hbase-1.3.1/bin/start-hbase.sh
```

启动日志：

```
hop03: starting regionserver, logging to /opt/hbase-1.3.1/bin/../logs/hbase-root-regionserver-hop03.out
hop02: starting regionserver, logging to /opt/hbase-1.3.1/bin/../logs/hbase-root-regionserver-hop02.out
hop01: starting regionserver, logging to /opt/hbase-1.3.1/bin/../logs/hbase-root-regionserver-hop01.out
```

**9、查看状态**

```
jps

HMaster：主节点
HRegionServer：分区节点
```

**10、停止集群**

在hop01节点停止即可。

```
/opt/hbase-1.3.1/bin/stop-hbase.sh
```

**11、查看界面**

```
http://hop01:16010
```

![](https://images.gitee.com/uploads/images/2022/0213/141505_42e752f8_5064118.png "01-1.png")

# 三、基础Shell命令

**1、切入客户端**

```
/opt/hbase-1.3.1/bin/hbase shell
```

**2、查看表**

```
hbase(main):002:0> list
```

**3、创建表**

```
hbase(main):003:0> create 'user','info'
0 row(s) in 2.7910 seconds
=> Hbase::Table - user
```

**4、查看表结构**

```
hbase(main):010:0> describe 'user'
```

**5、添加数据**

```
put 'user','id01','info:name','tom'
put 'user','id01','info:age','18'
put 'user','id01','info:sex','male'
put 'user','id02','info:name','jack'
put 'user','id02','info:age','20'
put 'user','id02','info:sex','female'
```

**6、查看表数据**

```
hbase(main):010:0> scan 'user'
ROW     COLUMN+CELL                                                                             
id01    column=info:age, timestamp=1594448524308, value=18                                      
id01    column=info:name, timestamp=1594448513534, value=tom                                    
id01    column=info:sex, timestamp=1594448530817, value=male                                    
id02    column=info:age, timestamp=1594448542631, value=20                                      
id02    column=info:name, timestamp=1594448536520, value=jack                                   
id02    column=info:sex, timestamp=1594448548005, value=female
```

这些表结构和数据会在集群之间自动同步。

**7、查询指定列**

```
hbase(main):012:0> get 'user','id01'
COLUMN      CELL                                                                                    
info:age    timestamp=1594448524308, value=18                                                       
info:name   timestamp=1594448513534, value=tom                                                       
info:sex    timestamp=1594448530817, value=male
```

**8、统计行数**

```
hbase(main):013:0> count 'user'
```

**9、删除行数据**

```
hbase(main):014:0> deleteall 'user','id02'
```

**10、清空表数据**

```
hbase(main):016:0> truncate 'user'
```

**11、删除表**

```
hbase(main):018:0> disable 'user'
hbase(main):019:0> drop 'user'
```

# 四、JDBC基础查询

**1、核心依赖**

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>1.3.1</version>
</dependency>
```

**2、基础配置**

这里连接zookeeper集群地址即可。

```
zookeeper:
  address: 集群地址Url，逗号分隔
```

编写HBase配置和常用工具方法。

```java
@Component
public class HBaseConfig {

    private static String address;
    private static final Object lock=new Object();
    public static Configuration configuration = null;
    public static ExecutorService executor = null;
    public static Connection connection = null;

    /**
     * 获取连接
     */
    public static Connection getConnection(){
        if(null == connection){
            synchronized (lock) {
                if(null == connection){
                    configuration = new Configuration();
                    configuration.set("hbase.zookeeper.quorum", address);
                    try {
                        executor = Executors.newFixedThreadPool(10);
                        connection = ConnectionFactory.createConnection(configuration, executor);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
        return connection;
    }

    /**
     * 获取 HBaseAdmin
     */
    public static HBaseAdmin getHBaseAdmin(){
        HBaseAdmin admin = null;
        try{
            admin = (HBaseAdmin)getConnection().getAdmin();
        }catch(Exception e){
            e.printStackTrace();
        }
        return admin;
    }
    /**
     * 获取 Table
     */
    public static Table getTable(TableName tableName) {
        Table table = null ;
        try{
            table = getConnection().getTable(tableName);
        }catch(Exception e){
            e.printStackTrace();
        }
        return table ;
    }
    /**
     * 关闭资源
     */
    public static void close(HBaseAdmin admin,Table table){
        try {
            if(admin!=null) {
                admin.close();
            }
            if(table!=null) {
                table.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Value("${zookeeper.address}")
    public void setAddress (String address) {
        HBaseConfig.address = address;
    }
}
```

**3、查询案例**

查询数据参考上述全表扫描结果：

```java
@RestController
public class HBaseController {

    /**
     * 扫描全表
     */
    @GetMapping("/scanTable")
    public String scanTable () throws Exception {
        Table table = HBaseConfig.getTable(TableName.valueOf("user"));
        try {
            ResultScanner resultScanner = table.getScanner(new Scan());
            for (Result result : resultScanner) {
                printResult(result);
            }
        } finally {
            HBaseConfig.close(null, table);
        }
        return "success";
    }

    /**
     * 根据RowKey扫描
     */
    @GetMapping("/scanRowKey")
    public void scanRowKey() throws Exception {
        String rowKey = "id02";
        Table table = HBaseConfig.getTable(TableName.valueOf("user"));
        try {
            Result result = table.get(new Get(rowKey.getBytes()));
            printResult(result);
        } finally {
            HBaseConfig.close(null, table);
        }
    }

    /**
     * 输出 Result
     */
    private void printResult (Result result){
        NavigableMap<byte[], NavigableMap<byte[], NavigableMap<Long, byte[]>>> map = result.getMap();
        Set<Map.Entry<byte[], NavigableMap<byte[], NavigableMap<Long, byte[]>>>> set = map.entrySet();
        for (Map.Entry<byte[], NavigableMap<byte[], NavigableMap<Long, byte[]>>> entry : set) {
            Set<Map.Entry<byte[], NavigableMap<Long, byte[]>>> entrySet = entry.getValue().entrySet();
            for (Map.Entry<byte[], NavigableMap<Long, byte[]>> entry2 : entrySet) {
                System.out.print(new String(result.getRow()));
                System.out.print("\t");
                System.out.print(new String(entry.getKey()));
                System.out.print(":");
                System.out.print(new String(entry2.getKey()));
                System.out.print(" value = ");
                System.out.println(new String(entry2.getValue().firstEntry().getValue()));
            }
        }
    }
}
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent/tree/master/series02-hbase-parent