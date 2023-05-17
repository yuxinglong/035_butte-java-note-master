# 一、JDK1.8 环境搭建

## 1、上传文件解压

```
[root@localhost mysoft]# tar -zxvf jdk-8u161-linux-x64.tar.gz
[root@localhost mysoft]# pwd
/usr/local/mysoft
[root@localhost mysoft]# mv jdk1.8.0_161 jdk1.8
```

## 2、检查环境，已经安装删除

```
[root@localhost mysoft]# ps -aux|grep java
[root@localhost mysoft]# rpm -e --nodeps rpm -qa | grep java
```

## 3、配置环境变量

```
[root@localhost /]# vim /etc/profile
# 文件末尾追加 下面内容 shit+g 跳到文件末尾
# JAVA_HOME
export JAVA_HOME=/usr/local/mysoft/jdk1.8
export PATH=$PATH:$JAVA_HOME/bin
```

## 4、检测安装成功

```
[root@localhost /]# java -version
openjdk version "1.8.0_161"
OpenJDK Runtime Environment (build 1.8.0_161-b14)
OpenJDK 64-Bit Server VM (build 25.161-b14, mixed mode)
```

# 二、TOMCAT8 安装

## 1、上传安装包

```
[root@localhost mysoft]# tar -zxvf apache-tomcat-8.5.40.tar.gz
[root@localhost mysoft]# mv apache-tomcat-8.5.40 tomcat8.5
```

## 2、启动服务

```
[root@localhost bin]# pwd
/usr/local/mysoft/tomcat8.5/bin
[root@localhost bin]# ./startup.sh 
Tomcat started.
```

## 3、访问测试

```
http://127.0.0.1:8080/ OK了
```

# 三、MySQL5.7 安装

## 1、卸载原系统中的mariadb

```
[root@localhost /]# rpm -qa|grep mariadb
mariadb-libs-5.5.56-2.el7.x86_64
[root@localhost /]# rpm -e --nodeps mariadb-libs
[root@localhost /]# rpm -qa|grep mariadb
```

## 2、获取官方地址

![](https://images.gitee.com/uploads/images/2022/0214/230922_a0489b72_5064118.png "02-1.png")

![](https://images.gitee.com/uploads/images/2022/0214/230932_bb1217c7_5064118.png "02-2.png")

```
地址：https://dev.mysql.com/downloads/repo/yum/
https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
```

## 3、Yum源安装

```
wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
[root@localhost /]# rpm -ivh mysql80-community-release-el7-3.noarch.rpm
[root@localhost /]# yum repolist all|grep mysql
mysql57-community/x86_64 MySQL 5.7 Community Server disabled
mysql80-community/x86_64 MySQL 8.0 Community Server enabled: 113
mysql80-community-source MySQL 8.0 Community Server - disabled
……..
```

yum源中默认启用的安装包版本为MySQL8.0，这里切换为5.7，执行以下命令；

```
[root@localhost /]# yum-config-manager --disable mysql80-community
[root@localhost /]# yum-config-manager --enable mysql57-community
```

![](https://images.gitee.com/uploads/images/2022/0214/230953_bbb883b6_5064118.png "02-3.png")

## 4、MySQL 安装启动

```
[root@localhost /]# yum install mysql-community-server
# 需要安装依赖提示，选择y
Total download size: 192 M
Installed size: 865 M
Is this ok [y/d/N]: y
```

查看版本

```
[root@localhost /]# mysql -V
mysql Ver 14.14 Distrib 5.7.26, for Linux (x86_64) using EditLine wrapper
```

启动查看状态

```
[root@localhost /]# systemctl start mysqld.service
[root@localhost /]# systemctl status mysqld.service 
	mysqld.service - MySQL Server
 Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
 Active: active (running) since Tue 2019-05-14 17:26:32 CST; 31s ago
```

为root账户生成临时密码

```
[root@localhost /]# grep 'temporary password' /var/log/mysqld.log
2019-05-14T09:26:28.657250Z 1 [Note] A temporary password is generated for root@localhost: !Bh(GT.od9L;
```

设置root用户密码

```
# 这里防止出现密码策略，强度不够的问题
mysql> set global validate_password_policy=LOW;
mysql> alter user 'root'@'localhost' identified by 'husky123456';
```

这样，Java的基础环境就搭建完毕了！

**源码参考：** https://gitee.com/cicadasmile/linux-system-base