# 一、DataX工具简介

## 1、设计理念

DataX是一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能。解决异构数据源同步问题，DataX将复杂的网状的同步链路变成了星型数据链路，DataX作为中间传输载体负责连接各种数据源。当需要接入一个新的数据源的时候，只需要将此数据源对接到DataX，便能跟已有的数据源做到无缝数据同步。

![](https://images.gitee.com/uploads/images/2022/0212/145122_38f4826a_5064118.png "06-1.png")

`絮叨一句`：异构数据源指，为了处理不同种类的业务，使用不同的数据库系统存储数据。

## 2、组件结构

DataX本身作为离线数据同步框架，采用Framework+plugin架构构建。将数据源读取和写入抽象成为Reader和Writer插件，纳入到整个同步框架中。

![](https://images.gitee.com/uploads/images/2022/0212/145139_b1510ce2_5064118.png "06-2.png")

- Reader

Reader为数据采集模块，负责读取采集数据源的数据，将数据发送给Framework。

- Writer

Writer为数据写入模块，负责不断向Framework取数据，并将数据写入到目的端。

- Framework

Framework用于连接reader和writer，作为两者的数据传输通道，并处理缓冲，流控，并发，数据转换等核心技术问题。

## 3、架构设计

![](https://images.gitee.com/uploads/images/2022/0212/145150_107afb85_5064118.png "06-3.png")

- Job

DataX完成单个数据同步的作业，称为Job，DataX接受到一个Job之后，将启动一个进程来完成整个作业同步过程。Job模块是单个作业的中枢管理节点，承担了数据清理、子任务切分(将单一作业计算转化为多个子Task)、TaskGroup管理等功能。

- Split

DataXJob启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。Task便是DataX作业的最小单元，每一个Task都会负责一部分数据的同步工作。

- Scheduler

切分多个Task之后，Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup(任务组)。

- TaskGroup

每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。每一个Task都由TaskGroup负责启动，Task启动后，会固定启动Reader—>Channel—>Writer的线程来完成任务同步工作。DataX作业运行起来之后，Job监控并等待多个TaskGroup模块任务完成，等待所有TaskGroup任务完成后Job成功退出。否则，异常退出，进程退出值非0。

# 二、环境安装

推荐Python2.6+，Jdk1.8+(脑补安装流程)。

## 1、Python包下载

```
# yum -y install wget
# wget https://www.python.org/ftp/python/2.7.15/Python-2.7.15.tgz
# tar -zxvf Python-2.7.15.tgz
```

## 2、安装Python

```
# yum install gcc openssl-devel bzip2-devel
[root@ctvm01 Python-2.7.15]# ./configure --enable-optimizations
# make altinstall
# python -V
```

## 3、DataX安装

```
# pwd
/opt/module
# ll
datax
# cd /opt/module/datax/bin
-- 测试环境是否正确
# python datax.py /opt/module/datax/job/job.json
```

# 三、同步任务

## 1、同步表创建

```sql
-- PostgreSQL
CREATE TABLE sync_user (
	id INT NOT NULL,
	user_name VARCHAR (32) NOT NULL,
	user_age int4 NOT NULL,
	CONSTRAINT "sync_user_pkey" PRIMARY KEY ("id")
);
CREATE TABLE data_user (
	id INT NOT NULL,
	user_name VARCHAR (32) NOT NULL,
	user_age int4 NOT NULL,
	CONSTRAINT "sync_user_pkey" PRIMARY KEY ("id")
);
```

## 2、编写任务脚本

```
[root@ctvm01 job]# pwd
/opt/module/datax/job
[root@ctvm01 job]# vim postgresql_job.json
```

## 3、脚本内容

```json
{
    "job": {
        "setting": {
            "speed": {
                "channel": "3"
            }
        },
        "content": [
            {
                "reader": {
                    "name": "postgresqlreader",
                    "parameter": {
                        "username": "root01",
                        "password": "123456",
                        "column": ["id","user_name","user_age"], 
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:postgresql://192.168.72.131:5432/db_01"], 
                                "table": ["data_user"]
                            }
                        ]
                    }
                }, 
                "writer": {
                    "name": "postgresqlwriter", 
                    "parameter": {
                        "username": "root01",
                        "password": "123456",
                        "column": ["id","user_name","user_age"], 
                        "connection": [
                            {
                                "jdbcUrl": "jdbc:postgresql://192.168.72.131:5432/db_01", 
                                "table": ["sync_user"]
                            }
                        ], 
                        "postSql": [], 
                        "preSql": []
                    }
                }
            }
        ]
    }
}
```

## 4、执行脚本

```
# /opt/module/datax/bin/datax.py /opt/module/datax/job/postgresql_job.json
```

## 5、执行日志

```
2020-04-23 18:25:33.404 [job-0] INFO  JobContainer - 
任务启动时刻                    : 2020-04-23 18:25:22
任务结束时刻                    : 2020-04-23 18:25:33
任务总计耗时                    :                 10s
任务平均流量                    :                1B/s
记录写入速度                    :              0rec/s
读出记录总数                    :                   2
读写失败总数                    :                   0
```

# 四、源码流程分析

**注意**：这里源码只贴出核心流程，如果要看完整源码，可以自行从Git上下载。

## 1、读取数据

核心入口：PostgresqlReader

启动读任务

```java
public static class Task extends Reader.Task {
    @Override
    public void startRead(RecordSender recordSender) {
        int fetchSize = this.readerSliceConfig.getInt(com.alibaba.datax.plugin.rdbms.reader.Constant.FETCH_SIZE);
        this.commonRdbmsReaderSlave.startRead(this.readerSliceConfig, recordSender,
                super.getTaskPluginCollector(), fetchSize);
    }
}
```

读取任务启动之后，执行读取数据操作。

核心类：CommonRdbmsReader

```java
public void startRead(Configuration readerSliceConfig,
                      RecordSender recordSender,
                      TaskPluginCollector taskPluginCollector, int fetchSize) {
    ResultSet rs = null;
    try {
        // 数据读取
        rs = DBUtil.query(conn, querySql, fetchSize);
        queryPerfRecord.end();
        ResultSetMetaData metaData = rs.getMetaData();
        columnNumber = metaData.getColumnCount();
        PerfRecord allResultPerfRecord = new PerfRecord(taskGroupId, taskId, PerfRecord.PHASE.RESULT_NEXT_ALL);
        allResultPerfRecord.start();
        long rsNextUsedTime = 0;
        long lastTime = System.nanoTime();
        // 数据传输至交换区
        while (rs.next()) {
            rsNextUsedTime += (System.nanoTime() - lastTime);
            this.transportOneRecord(recordSender, rs,metaData, columnNumber, mandatoryEncoding, taskPluginCollector);
            lastTime = System.nanoTime();
        }
        allResultPerfRecord.end(rsNextUsedTime);
    }catch (Exception e) {
        throw RdbmsException.asQueryException(this.dataBaseType, e, querySql, table, username);
    } finally {
        DBUtil.closeDBResources(null, conn);
    }
}
```

## 2、数据传输

核心接口：RecordSender(发送)

```java
public interface RecordSender {
	public Record createRecord();
	public void sendToWriter(Record record);
	public void flush();
	public void terminate();
	public void shutdown();
}
```

核心接口：RecordReceiver(接收)

```java
public interface RecordReceiver {
	public Record getFromReader();
	public void shutdown();
}
```

核心类：BufferedRecordExchanger

```java
class BufferedRecordExchanger implements RecordSender, RecordReceiver
```

## 3、写入数据

核心入口：PostgresqlWriter

启动写任务

```java
public static class Task extends Writer.Task {
	public void startWrite(RecordReceiver recordReceiver) {
		this.commonRdbmsWriterSlave.startWrite(recordReceiver, this.writerSliceConfig, super.getTaskPluginCollector());
	}
}
```

写数据任务启动之后，执行数据写入操作。

核心类：CommonRdbmsWriter

```java
public void startWriteWithConnection(RecordReceiver recordReceiver,
                                     Connection connection) {
    // 写数据库的SQL语句
    calcWriteRecordSql();
    List<Record> writeBuffer = new ArrayList<>(this.batchSize);
    int bufferBytes = 0;
    try {
        Record record;
        while ((record = recordReceiver.getFromReader()) != null) {
            writeBuffer.add(record);
            bufferBytes += record.getMemorySize();
            if (writeBuffer.size() >= batchSize || bufferBytes >= batchByteSize) {
                doBatchInsert(connection, writeBuffer);
                writeBuffer.clear();
                bufferBytes = 0;
            }
        }
        if (!writeBuffer.isEmpty()) {
            doBatchInsert(connection, writeBuffer);
            writeBuffer.clear();
            bufferBytes = 0;
        }
    } catch (Exception e) {
        throw DataXException.asDataXException(
                DBUtilErrorCode.WRITE_DATA_ERROR, e);
    } finally {
        writeBuffer.clear();
        bufferBytes = 0;
        DBUtil.closeDBResources(null, null, connection);
    }
}
```

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap01-data-source/case06-datax-sync