# 一、MinIO简介

## 1、基础描述

MinIO是一个开源的对象存储服务。适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

MinIO是一个非常轻量的服务,可以很简单的和其他应用的结合，类似 NodeJS, Redis 或者 MySQL。

## 2、存储机制

MinIO使用按对象的嵌入式擦除编码保护数据，该编码以汇编代码编写，可提供最高的性能。MinIO使用Reed-Solomon代码将对象划分为n/2个数据和n / 2个奇偶校验块-尽管可以将它们配置为任何所需的冗余级别。 这意味着在12个驱动器设置中，将一个对象分片为6个数据和6个奇偶校验块。即使丢失了多达5个(（n/2）–1)个驱动器（无论是奇偶校验还是数据），仍然可以从其余驱动器可靠地重建数据。MinIO的实现可确保即使丢失或无法使用多个设备，也可以读取对象或写入新对象。最后，MinIO的擦除代码位于对象级别，并且可以一次修复一个对象。

# 二、MinIO环境搭建

## 1、安装包下载

```
https://dl.min.io/server/minio/release/linux-amd64/minio
```

建议使用某雷下载，速度会快点，下载包上传到`/opt/minioconfig/run`目录下。

## 2、创建数据存储目录

```
mkdir -p /data/minio/data
```

## 3、服务启动

启动并指定数据存放地址

```
/opt/minioconfig/run/minio server /data/minio/data/
```

输出日志

```
Endpoint:  http://localhost:9000  http://127.0.0.1:9000    
AccessKey: minioadmin 
SecretKey: minioadmin
```

这里就是登录地址和账号密码。

# 三、整合SpringBoot环境

## 1、基础依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>3.0.12</version>
</dependency>
```

## 2、基础配置

配置要素：地址和端口，登录名，密码，HTML存储桶，图片存储桶。

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/152219_302b6244_5064118.jpeg "24-1.jpg")

```yaml
minio:
  endpoint: http://192.168.72.133:9000
  accessKey: minioadmin
  secretKey: minioadmin
  bucketNameHtml: html
  bucketNameImage: image
```

文件上传之后，可以基于文件地址直接访问，但是需要在MinIO中配置文件的读写权限：

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/152248_7964bbdb_5064118.jpeg "24-2.jpg")

## 3、配置参数类

```java
@Component
@ConfigurationProperties(prefix = "minio")
public class ParamConfig {

    private String endpoint ;
    private String accessKey ;
    private String secretKey ;
    private String bucketNameHtml ;
    private String bucketNameImage ;
    // 省略 get 和 set方法
}
```

## 4、基于MinIO配置类

封装MinIO客户端连接工具，文件上传的基础方法，返回文件在MinIO服务上的URL地址。

```java
import io.minio.MinioClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import javax.annotation.PostConstruct;
import javax.annotation.Resource;

@Component
public class MinIOConfig {

    private static final Logger LOGGER = LoggerFactory.getLogger(MinIOConfig.class) ;

    @Resource
    private ParamConfig paramConfig ;

    private MinioClient minioClient ;

    /**
     * 初始化 MinIO 客户端
     */
    @PostConstruct
    private void init(){
        try {
            minioClient = new MinioClient(paramConfig.getEndpoint(),
                                          paramConfig.getAccessKey(),
                                          paramConfig.getSecretKey());
        } catch (Exception e) {
            e.printStackTrace();
            LOGGER.info("MinIoClient init fail ...");
        }
    }

    /**
     * 上传 <html> 页面
     */
    public String uploadHtml (String fileName, String filePath) throws Exception {
        minioClient.putObject(paramConfig.getBucketNameHtml(),fileName,filePath);
        return paramConfig.getEndpoint()+"/"+paramConfig.getBucketNameHtml()+"/"+fileName ;
    }

    /**
     * 上传 <img> 图片
     */
    public String uploadImg (String imgName, String imgPath) throws Exception {
        minioClient.putObject(paramConfig.getBucketNameImage(),imgName,imgPath);
        return paramConfig.getEndpoint()+"/"+paramConfig.getBucketNameImage()+"/"+imgName ;
    }
}
```

## 5、服务实现

提供两个基础方法：HTML和图片上传，存储在不同位置。

```java
import com.minio.file.config.MinIOConfig;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;

@Service
public class UploadServiceImpl implements UploadService {

    @Resource
    private MinIOConfig minIOConfig ;

    // 上传 <html> ,返回服务器地址
    @Override
    public String uploadHtml(String fileName, String filePath) throws Exception {
        return minIOConfig.uploadHtml(fileName,filePath);
    }

    // 上传 <img> ,返回服务器地址
    @Override
    public String uploadImg(String imgName, String imgPath) throws Exception {
        return minIOConfig.uploadImg(imgName,imgPath);
    }
}
```

上传之后，基于浏览器访问接口返回的url，查看效果：

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/163052_ee89de33_5064118.jpeg "24-3.jpg")

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware23-minio-file