# 一、环境和版本

Linux：centos7 三台

```
三台Linux服务
192.168.72.129
192.168.72.130
192.168.72.131
Redis：redis-4.0.14
```

# 二、上传Redis软件

## 1、创建软件目录

```
[root@localhost local]# cd /usr/local/
[root@localhost local]# mkdir mysoft
```

## 2、Xftp上传软件，解压

```
[root@localhost mysoft]# cd /usr/local/mysoft/
[root@localhost mysoft]# ll
total 1704
-rw-r--r--. 1 root root 1740967 Apr 30 11:29 redis-4.0.14.tar.gz
[root@localhost mysoft]# tar -zxvf redis-4.0.14.tar.gz
```

## 3、编译项目

```
[root@localhost mysoft]# ll
total 1708
drwxrwxr-x. 6 root root 4096 Mar 19 00:23 redis-4.0.14
-rw-r--r--. 1 root root 1740967 Apr 30 11:29 redis-4.0.14.tar.gz
[root@localhost mysoft]# cd redis-4.0.14/
[root@localhost redis-4.0.14]# make MALLOC=libc
```

## 4、安装Redis

```
[root@localhost redis-4.0.14]# cd src && make install
```

## 5、启动服务

```
[root@localhost redis-4.0.14]# cd src
[root@localhost src]# ./redis-server
```

## 6、配置进程启动

修改redis.conf

```
daemonize yes
```

## 7、进程查看关闭

```
[root@localhost redis-4.0.14]# ./src/redis-server redis.conf
11320:C 05 May 14:26:31.053 # Redis is starting
11320:C 05 May 14:26:31.053 # Redis version=4.0.14, bits=64, commit=00000000, modified=0, pid=11320, just started
11320:C 05 May 14:26:31.053 # Configuration loaded
[root@localhost redis-4.0.14]# ps -aux |grep redis
root 11321 0.1 0.1 141840 2028 ? Ssl 14:26 0:00 ./src/redis-server *:6379
root 11338 0.0 0.0 112708 980 pts/1 S+ 14:27 0:00 grep --color=auto redis
[root@localhost redis-4.0.14]# kill -9 11321
```

# 三、配置开机启动

## 1、相关配置

```
[root@localhost init.d]# cd /etc
[root@localhost etc]# mkdir redis
[root@localhost etc]# cp /usr/local/mysoft/redis-4.0.14/redis.conf /etc/redis/6379.conf
[root@localhost etc]# cd redis/
[root@localhost redis]# ll
total 60
-rw-r--r--. 1 root root 58767 May 5 14:36 redis-6379.conf
[root@localhost redis]# cp /usr/local/mysoft/redis-4.0.14/utils/redis_init_script /etc/init.d/redisd
[root@localhost redis]# chkconfig redisd on # 开机启动命令
```

## 2、服务启动关闭

```
[root@localhost redis]# service redisd start
Starting Redis server...
3163:C 05 May 14:59:13.872 # Redis is starting 
3163:C 05 May 14:59:13.872 # Redis version=4.0.14, bits=64, commit=00000000, modified=0, pid=3163, just started
3163:C 05 May 14:59:13.872 # Configuration loaded
[root@localhost redis]# service redisd stop
Stopping ...
Waiting for Redis to shutdown ...
Redis stopped
```

## 3、重启虚拟机查看Redis状态

```
[root@localhost ~]# ps -aux |grep redis
root 987 0.1 0.1 141836 2012 ? Ssl 15:02 0:00 /usr/local/bin/redis-server *:6379
root 2966 0.0 0.0 112712 980 pts/1 S+ 15:04 0:00 grep --color=auto redis
```

# 四、解决客户端连接问题

关闭防火墙，或者开放6379端口

```
firewalld的基本使用
启动： systemctl start firewalld
关闭： systemctl stop firewalld
查看状态： systemctl status firewalld
开机禁用 ： systemctl disable firewalld
开机启用 ： systemctl enable firewalld
```

修改redis.conf 配置

```
注释掉：# bind 127.0.0.1
修改保护模式：protected-mode no
```

# 五、sentinel哨兵模式

## 1、基础配置

```
192.168.72.129 主服务
192.168.72.130 从服务
192.168.72.131 从服务
```

## 2、配置主服务 redis.conf

```
requirepass 123456
masterauth 123456
```

## 3、配置从服务 redis.conf

```
requirepass 123456
slaveof 192.168.72.129 6379
masterauth 123456
```

## 4、配置sentinel.conf

```
protected-mode no
# sentinel monitor代表监控
# mymaster代表服务器的名称，可以自定义，
# 192.168.72.129代表监控的主服务器，6379代表端口，
# 2 标识 >=2 哨兵认为主服务器不可用，执行failover操作。
sentinel monitor mymaster 192.168.72.129 6379 2
sentinel auth-pass mymaster 123456
```

## 5、启动服务

先主服务，后从服务

```
[root@localhost src]# ./redis-server ../redis.conf
[root@localhost src]# ./redis-sentinel ../sentinel.conf
```

没错，就是这样搭建完毕了！

**源码参考：** https://gitee.com/cicadasmile/linux-system-base