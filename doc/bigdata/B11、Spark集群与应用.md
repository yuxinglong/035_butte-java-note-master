# 一、Spark概述

## 1、Spark简介

Spark是专为大规模数据处理而设计的，基于内存快速通用,可扩展的集群计算引擎，实现了高效的DAG执行引擎,可以通过基于内存来高效处理数据流，运算速度相比于MapReduce得到了显著的提高。

##  2、运行结构

![](https://images.gitee.com/uploads/images/2022/0213/152956_aca55513_5064118.png "01-1.png")

**Driver**

运行Spark的Applicaion中main()函数，会创建SparkContext，SparkContext负责和Cluster-Manager进行通信，并负责申请资源、任务分配和监控等。

**ClusterManager**

负责申请和管理在WorkerNode上运行应用所需的资源，可以高效地在一个计算节点到数千个计算节点之间伸缩计算，目前包括Spark原生的ClusterManager、ApacheMesos和HadoopYARN。

**Executor**

Application运行在WorkerNode上的一个进程，作为工作节点负责运行Task任务，并且负责将数据存在内存或者磁盘上，每个 Application都有各自独立的一批Executor，任务间相互独立。

# 二、环境部署

## 1、Scala环境

**安装包管理**

```
[root@hop01 opt]# tar -zxvf scala-2.12.2.tgz
[root@hop01 opt]# mv scala-2.12.2 scala2.12
```

**配置变量**

```
[root@hop01 opt]# vim /etc/profile

export SCALA_HOME=/opt/scala2.12
export PATH=$PATH:$SCALA_HOME/bin

[root@hop01 opt]# source /etc/profile
```

**版本查看**

```
[root@hop01 opt]# scala -version
```

Scala环境需要部署在Spark运行的相关服务节点上。

## 2、Spark基础环境

**安装包管理**

```
[root@hop01 opt]# tar -zxvf spark-2.1.1-bin-hadoop2.7.tgz
[root@hop01 opt]# mv spark-2.1.1-bin-hadoop2.7 spark2.1
```

**配置变量**

```
[root@hop01 opt]# vim /etc/profile

export SPARK_HOME=/opt/spark2.1
export PATH=$PATH:$SPARK_HOME/bin

[root@hop01 opt]# source /etc/profile
```

**版本查看**

```
[root@hop01 opt]# spark-shell
```

![](https://images.gitee.com/uploads/images/2022/0213/153010_695376c8_5064118.png "01-2.png")

## 3、Spark集群配置

**服务节点**

```
[root@hop01 opt]# cd /opt/spark2.1/conf/
[root@hop01 conf]# cp slaves.template slaves
[root@hop01 conf]# vim slaves

hop01
hop02
hop03
```

**环境配置**

```
[root@hop01 conf]# cp spark-env.sh.template spark-env.sh
[root@hop01 conf]# vim spark-env.sh

export JAVA_HOME=/opt/jdk1.8
export SCALA_HOME=/opt/scala2.12
export SPARK_MASTER_IP=hop01
export SPARK_LOCAL_IP=安装节点IP
export SPARK_WORKER_MEMORY=1g
export HADOOP_CONF_DIR=/opt/hadoop2.7/etc/hadoop
```

注意`SPARK_LOCAL_IP`的配置。

## 4、Spark启动

依赖Hadoop相关环境，所以要先启动。

```
启动：/opt/spark2.1/sbin/start-all.sh
停止：/opt/spark2.1/sbin/stop-all.sh
```

这里在主节点会启动两个进程：Master和Worker，其他节点只启动一个Worker进程。

## 5、访问Spark集群

默认端口是：8080。

```
http://hop01:8080/
```

![](https://images.gitee.com/uploads/images/2022/0213/153021_3e359432_5064118.png "01-3.png")

运行基础案例：

```
[root@hop01 spark2.1]# cd /opt/spark2.1/
[root@hop01 spark2.1]# bin/spark-submit --class org.apache.spark.examples.SparkPi --master local examples/jars/spark-examples_2.11-2.1.1.jar

运行结果：Pi is roughly 3.1455357276786384
```

# 三、开发案例

## 1、核心依赖

依赖Spark2.1.1版本：

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-core_2.11</artifactId>
    <version>2.1.1</version>
</dependency>
```

引入Scala编译插件：

```xml
<plugin>
    <groupId>net.alchim31.maven</groupId>
    <artifactId>scala-maven-plugin</artifactId>
    <version>3.2.2</version>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>testCompile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## 2、案例代码开发

读取指定位置的文件，并输出文件内容单词统计结果。

```java
@RestController
public class WordWeb implements Serializable {

    @GetMapping("/word/web")
    public String getWeb (){
        // 1、创建Spark的配置对象
        SparkConf sparkConf = new SparkConf().setAppName("LocalCount")
                                             .setMaster("local[*]");

        // 2、创建SparkContext对象
        JavaSparkContext sc = new JavaSparkContext(sparkConf);
        sc.setLogLevel("WARN");

        // 3、读取测试文件
        JavaRDD lineRdd = sc.textFile("/var/spark/test/word.txt");

        // 4、行内容进行切分
        JavaRDD wordsRdd = lineRdd.flatMap(new FlatMapFunction() {
            @Override
            public Iterator call(Object obj) throws Exception {
                String value = String.valueOf(obj);
                String[] words = value.split(",");
                return Arrays.asList(words).iterator();
            }
        });

        // 5、切分的单词进行标注
        JavaPairRDD wordAndOneRdd = wordsRdd.mapToPair(new PairFunction() {
            @Override
            public Tuple2 call(Object obj) throws Exception {
                //将单词进行标记：
                return new Tuple2(String.valueOf(obj), 1);
            }
        });

        // 6、统计单词出现次数
        JavaPairRDD wordAndCountRdd = wordAndOneRdd.reduceByKey(new Function2() {
            @Override
            public Object call(Object obj1, Object obj2) throws Exception {
                return Integer.parseInt(obj1.toString()) + Integer.parseInt(obj2.toString());
            }
        });

        // 7、排序
        JavaPairRDD sortedRdd = wordAndCountRdd.sortByKey();
        List<Tuple2> finalResult = sortedRdd.collect();

        // 8、结果打印
        for (Tuple2 tuple2 : finalResult) {
            System.out.println(tuple2._1 + " ===> " + tuple2._2);
        }

        // 9、保存统计结果
        sortedRdd.saveAsTextFile("/var/spark/output");
        sc.stop();
        return "success" ;
    }
}
```

打包执行结果：

![](https://images.gitee.com/uploads/images/2022/0213/153032_533326ce_5064118.png "01-4.png")

查看文件输出：

```
[root@hop01 output]# vim /var/spark/output/part-00000
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent/tree/master/series03-spark-parent