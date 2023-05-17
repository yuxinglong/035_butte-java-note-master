# 一、环境搭建

环境版本

```
centos7
jdk1.8 已搭建好
rocketmq4.3
```

## 1、下载安装包

网址

```
https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.3.2/rocketmq-all-4.3.2-bin-release.zip
# We suggest the following mirror site for your download:官方建议下载地址
http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.3.2/rocketmq-all-4.3.2-bin-release.zip
```

## 2、上传文件

```
[root@localhost mysoft]# pwd
/usr/local/mysoft
[root@localhost mysoft]# unzip rocketmq-all-4.3.2-bin-release.zip
[root@localhost mysoft]# mv rocketmq-all-4.3.2-bin-release rocket4.3
[root@localhost mysoft]# rm -f rocketmq-all-4.3.2-bin-release.zip
```

## 3、修改相关配置

rocketmq的默认配置极其耗内存，要进行修改。

1）修改runserver.sh配置

注释掉原来的，添加新配置

```
[root@localhost bin]# vim runserver.sh
#JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn512m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

2）修改runbroker.sh配置

注释掉原来的，添加新配置

```
[root@localhost bin]# vim runbroker.sh
#JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m"
```

3）修改tools.sh配置

注释掉原来的，添加新配置

```
[root@localhost bin]# vim tools.sh
#JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn256m -XX:PermSize=128m -XX:MaxPermSize=128m"
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn256m -XX:PermSize=128m -XX:MaxPermSize=128m"
```

## 4、启动服务

要按照顺序启动

```
nohup sh /usr/local/mysoft/rocket4.3/bin/mqnamesrv
# 指定端口
nohup sh /usr/local/mysoft/rocket4.3/bin/mqbroker -n localhost:9876
```

# 二、监控台搭建

## 1、git包下载配置（Win10系统）

```
地址：https://github.com/apache/rocketmq-externals.git 
下载完成之后，进入
rocketmq-externals\rocketmq-console\src\main\resources
文件夹，打开
application.properties
修改如下配置：
server.port=8089
rocketmq.config.namesrvAddr=192.168.72.129:9876
```

## 2、执行打包操作（Win10系统）

```
进入如下目录，打开命令行
rocketmq-externals\rocketmq-console
执行打包命令
mvn clean package -Dmaven.test.skip=true
编译生成。
rocketmq-console-ng-1.0.0.jar
```

## 3、上传jar包到Linux服务

```
[root@localhost myjar]# pwd
/usr/local/myjar
[root@localhost myjar]# ll
-rw-r--r--. 1 root root 33231510 May 16 11:11 rocketmq-console-ng-1.0.0.jar
```

## 4、启动监控台

```
[root@localhost myjar]# java -jar rocketmq-console-ng-1.0.0.jar
```

## 5、测试安装结果

浏览器访问

```
http://192.168.72.129:8089
```

![](https://images.gitee.com/uploads/images/2022/0214/231821_e32052e1_5064118.png "05-1.png")

**源码参考：** https://gitee.com/cicadasmile/linux-system-base