# 一、FastDFS简介

## 1、基础概念

FastDFS是一个开源的轻量级分布式文件系统，它对文件进行管理，功能包括：文件存储、文件同步、文件上传、文件下载等，解决了大容量存储和负载均衡的问题。

## 2、环境概览

```
1、默认存在Gcc编译环境，Centos7虚拟机
2、安装LibFastCommon环境
3、FastDFS中间件安装
4、Nginx代理服务器安装
```

# 二、安装LibFastCommon

核心流程

下载->解压->编译->安装

```
## 下载
[root@localhost mysoft]# wget
https://github.com/happyfish100/libfastcommon/archive/V1.0.38.tar.gz
## 解压
[root@localhost mysoft]# tar -zxvf V1.0.38.tar.gz
[root@localhost mysoft]# cd libfastcommon-1.0.38/
## 编译
[root@localhost libfastcommon-1.0.38]# ./make.sh
## 安装
[root@localhost libfastcommon-1.0.38]# ./make.sh install
```

# 三、安装FastDFS

流程：下载->解压->编译->安装->创建相关路径->配置跟踪器->
      配置数据存储->配置客户端->Nginx环境配置
      
## 1、基础安装步骤

```
## 下载
[root@localhost mysoft]# wget
https://github.com/happyfish100/fastdfs/archive/V5.11.tar.gz
## 解压
[root@localhost mysoft]# tar -zxvf V5.11.tar.gz 
## 编译
[root@localhost mysoft]# cd fastdfs-5.11/
[root@localhost fastdfs-5.11]# ./make.sh 
## 安装
[root@localhost fastdfs-5.11]# ./make.sh install
```

## 2、创建相关路径

用处后续说明。

```
[root@localhost data]# mkdir -p /data/fastdfs/log
[root@localhost data]# mkdir -p /data/fastdfs/data
[root@localhost data]# mkdir -p /data/fastdfs/tracker
[root@localhost data]# mkdir -p /data/fastdfs/client
```

## 3、配置跟踪器

Tracker -- >> 跟踪器

1）查看配置文件

注意这里目录的转换，这里给的是样例，具体的配置还要自己动手。

```
[root@localhost fastdfs-5.11]# cd /etc/fdfs/
[root@localhost fdfs]# ll
total 24
client.conf.sample
storage.conf.sample
storage_ids.conf.sample
tracker.conf.sample
```

2）配置tracker.conf文件

```
[root@localhost fdfs]# cp tracker.conf.sample tracker.conf
[root@localhost fdfs]# vim tracker.conf
## 关注如下几个配置
## 存储数据和日志文件的基本路径
base_path=/data/fastdfs/tracker
## Http服务端口
http.server_port=80
## 默认提供服务端口
port=22122
```

3）启动跟踪器

```
## 启动
[root@localhost fdfs]# /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf start
## 查看状态
[root@localhost fdfs]# netstat -apn|grep fdfs
```

## 4、配置数据存储

1）查看配置文件

```
[root@localhost fastdfs-5.11]# cd /etc/fdfs/
[root@localhost fdfs]# ll
storage.conf.sample
```

2）配置storage.conf文件

```
[root@localhost fdfs]# cp storage.conf.sample storage.conf
[root@localhost fdfs]# vim storage.conf
## 关注如下几个配置
## storage存储data和log的跟路径
base_path=/data/fastdfs/data
## 默认组名
group_name=group1
## 默认端口，相同组的storage端口号必须一致
port=23000
## 配置一个存储路径
store_path_count=1
store_path0=/data/fastdfs/data
## 配置跟踪器IP和端口
tracker_server=192.168.72.130:22122
```

3）启动存储服务

```
## 启动
[root@localhost fdfs]# /usr/bin/fdfs_storaged /etc/fdfs/storage.conf start
## 查看进程
[root@localhost fdfs]# netstat -apn|grep fdfs
tcp 0:22122  LISTEN      4845/fdfs_trackerd  
tcp 0:45422  SYN_SENT    5410/fdfs_storaged
## 查看启动日志
[root@localhost fdfs]# tail -f /data/fastdfs/data/logs/storaged.log
## 日志展示：单台FastDFS安装成功
set tracker leader: 192.168.72.130:22122
## 查看Storage和Tracker是否在通信
[root@localhost fdfs]# /usr/bin/fdfs_monitor /etc/fdfs/storage.conf
Storage 1:
	id = 192.168.72.130
	ip_addr = 192.168.72.130 (localhost.localdomain)  ACTIVE
```

## 5、配置客户端测试

1）查看配置文件

```
[root@localhost /]# cd /etc/fdfs
[root@localhost fdfs]# ll
total 40
client.conf.sample
```

2）配置client.conf文件

```
[root@localhost fdfs]# cp client.conf.sample client.conf
[root@localhost fdfs]# vim client.conf
## 关注如下几个配置
## client数据和日志目录
base_path=/data/fastdfs/client
## 配置跟踪器IP和端口
tracker_server=192.168.72.130:22122
```

3）客户端测试

调用客户端文件上传命令

/usr/bin/fdfs_upload_file /etc/fdfs/client.conf

返回文件上传的相对路径和编号

group1/M00/00/00/wKhIgl0mmE-ATEXPAAQ2pIoAy98392.jpg

```
[root@localhost fdfs]# /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /data/img/img1.jpg 
group1/M00/00/00/wKhIgl0mmE-ATEXPAAQ2pIoAy98392.jpg
```

这样FastDFS单台环境就安装好了，步骤有点繁杂，不过这就是生活。

文件成功上传storage服务器，但是还无法查看下载。需要安装Nginx服务器用来支持Http方式访问文件。

# 四、安装Nginx

## 1、下载解压Nginx

```
## 下载nginx
[root@localhost mysoft]# wget 
http://nginx.org/download/nginx-1.15.2.tar.gz
## 解压nginx
[root@localhost mysoft]# tar -zxvf nginx-1.15.2.tar.gz
```

## 2、下载解压Fast-Nginx

```
## 下载fastdfs-nginx
[root@localhost mysoft]#wget 
https://github.com/happyfish100/fastdfs-nginx-module/archive/5e5f3566bbfa57418b5506aaefbe107a42c9fcb1.zip
## 解压fastdfs-nginx
[root@localhost mysoft]# mv 5e5f3566bbfa57418b5506aaefbe107a42c9fcb1.zip fast-nginx.zip
[root@localhost mysoft]# unzip fast-nginx.zip
[root@localhost mysoft]# mv fastdfs-nginx-module-5e5f3566bbfa57418b5506aaefbe107a42c9fcb1/ 
fastdfs-nginx-module
```

## 3、安装必须依赖

```
## pcre-devel 环境
[root@localhost nginx-1.15.2]# yum install -y pcre pcre-devel
## zlib-devel 环境
[root@localhost nginx-1.15.2]# yum install -y zlib zlib-devel
## openssl-devel 环境
[root@localhost nginx-1.15.2]# yum install -y openssl openssl-devel
```

## 4、配置安装

```
[root@localhost nginx-1.15.2]# ./configure --add-module=/usr/local/mysoft/fastdfs-nginx-module/src
[root@localhost nginx-1.15.2]# make && make install
```

## 5、错误解决

版本问题导致，Fast-Nginx必须使用这个修复版本。

https://github.com/happyfish100/fastdfs-nginx-module/archive/5e5f3566bbfa57418b5506aaefbe107a42c9fcb1.zip
```
make[1]: *** [objs/addon/src/ngx_http_fastdfs_module.o] Error 1
make[1]: Leaving directory `/usr/local/mysoft/nginx-1.15.2'
make: *** [build] Error 2
```

## 6、查看安装结果

如下情况则表示安装成功了。

```
[root@localhost nginx-1.15.2]# /usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.15.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
configure arguments: --add-module=/usr/local/mysoft/fastdfs-nginx-module/src
```

# 五、测试图片访问

## 1、配置客户端

```
## 移动配置文件
[root@localhost src]# pwd
/usr/local/mysoft/fastdfs-nginx-module/src
[root@localhost src]# ll
total 76
Apr 14  2017 mod_fastdfs.conf
[root@localhost src]# cp mod_fastdfs.conf /etc/fdfs/
[root@localhost fdfs]# pwd
/etc/fdfs
[root@localhost fdfs]# vim mod_fastdfs.conf 
## 调整如下配置
## 链接超时
connect_timeout=20
## 配置跟踪器IP和端口
tracker_server=192.168.72.130:22122
## 路径包含group
url_have_group_name = true
# 必须和storage配置相同
store_path0=/data/fastdfs/data
```

## 2、完善FastDFS配置

```
[root@localhost fdfs]# cd /usr/local/mysoft/fastdfs-5.11/conf/
[root@localhost conf]# cp anti-steal.jpg http.conf mime.types /etc/fdfs/
```

## 3、配置Nginx

在Nginx的80服务端口下添加如下配置。注意这里的路径是Nginx安装自动生成的路径。

```
[root@localhost nginx]# cd /usr/local/nginx/conf/
[root@localhost conf]# vim nginx.conf
server {
    listen       80;
    
    location ~/group([0-9])/M00 {
        root /data/fastdfs/data;
        ngx_fastdfs_module;
    }
}
```

查看配置结果

```
[root@localhost conf]# /usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.15.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
configure arguments: --add-module=/usr/local/mysoft/fastdfs-nginx-module/src
```

这样就配置成功了。

## 4、见证结束的时候

启动Nginx服务。

```
## 启动
/usr/local/nginx/sbin/nginx
## 停止
/usr/local/nginx/sbin/nginx -s stop
## 重启
/usr/local/nginx/sbin/nginx -s reload
```

## 5、访问上传图片

```
http://192.168.72.130
/group1/M00/00/00/wKhIgl0mmE-ATEXPAAQ2pIoAy98392.jpg
```

**源码参考：** https://gitee.com/cicadasmile/linux-system-base