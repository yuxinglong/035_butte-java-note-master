# 一、NoSQL简介

## 1、NoSQL 概念

NoSQL(Not-Only-SQL )，意即"不仅仅是SQL"。对不同于传统的关系型数据库的数据库管理系统的统称。NoSQL用于超大规模数据的存储。这些类型的数据存储不需要固定的模式，无需多余操作就可以横向扩展。

## 2、NoSQL的优点/缺点

- 优点
    
    - 高可扩展性
    - 分布式计算
    - 低成本
    - 架构的灵活性，半结构化数据
    - 没有复杂的关系

- 缺点

    - 没有标准化
    - 有限的查询功能
    - 数据展现不直观

# 二、MongoDB数据库

## 1、MongoDB简介

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。他支持的数据结构非常松散，是类似 json 的 bjson 格式，因此可以存储比较复杂的数据类型。MongoDB 最大的特点是他支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

## 2、MongoDB特点

- MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。
- 在高负载的情况下，添加更多的节点，可以保证服务器性能。
- MongoDB 旨在为WEB应用提供可扩展的高性能数据存储解决方案。
- MongoDB 将数据存储为一个文档，数据结构由键值(key=&gt;value)对组成。MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

# 三、SpringBoot2整合

## 1、MongoDB基础环境

```
# 打开命令行
MongoDB4.0\bin&gt;mongo
# 展示所有数据库
&gt; show databases
# 新建一个admin数据库，命令比较难为情
&gt; db.admin.insert({"name":"管理员数据库"});
# 使用admin数据库
&gt; use admin
# 创建root用户，具有读写权限
&gt; db.createUser({user:"root",pwd:"root",roles:[{role:"readWrite",db:"admin"}]})
  Successfully added user:
```

## 2、核心依赖

```xml
<dependency>
    <groupid>org.springframework.boot</groupid>
    <artifactid>spring-boot-starter-data-mongodb</artifactid>
</dependency>
```

## 3、配置文件

- 用户名：root
- 密码：root
- 数据库：admin

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://root:root@localhost:27017/admin
```
## 4、封装应用接口

```java
public interface ImgInfoRepository {
    void saveImg(ImgInfo imgInfo) ;
    ImgInfo findByImgTitle(String imgTitle);
    long updateImgInfo(ImgInfo imgInfo) ;
    void deleteById(Integer imgId);
}
```

## 5、核心代码块

MongoDB的使用方式如下:

```java
import com.boot.mongodb.entity.ImgInfo;
import com.boot.mongodb.repository.ImgInfoRepository;
import com.mongodb.client.result.UpdateResult;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;
@Service
public class ImgInfoRepositoryImpl implements ImgInfoRepository {
    @Resource
    private MongoTemplate mongoTemplate;
    @Override
    public void saveImg(ImgInfo imgInfo) {
        mongoTemplate.save(imgInfo) ;
    }
    @Override
    public ImgInfo findByImgTitle(String imgTitle) {
        Query query=new Query(Criteria.where("imgTitle").is(imgTitle));
        return mongoTemplate.findOne(query,ImgInfo.class);
    }
    @Override
    public long updateImgInfo(ImgInfo imgInfo) {
        Query query = new Query(Criteria.where("imgId").is(imgInfo.getImgId()));
        Update update= new Update().set("imgTitle", imgInfo.getImgTitle()).set("imgUrl", imgInfo.getImgUrl());
        UpdateResult result = mongoTemplate.updateFirst(query,update,ImgInfo.class);
        return result.getMatchedCount();
    }
    @Override
    public void deleteById(Integer imgId) {
        Query query = new Query(Criteria.where("imgId").is(imgId));
        mongoTemplate.remove(query,ImgInfo.class);
    }
}
```

## 6、测试代码块

```java
import com.boot.mongodb.MongoDBApplication;
import com.boot.mongodb.entity.ImgInfo;
import com.boot.mongodb.repository.ImgInfoRepository;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import javax.annotation.Resource;
import java.util.Date;
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = MongoDBApplication.class)
public class MongoTest {
    @Resource
    private ImgInfoRepository imgInfoRepository ;
    @Test
    public void test1 (){
        ImgInfo record = new ImgInfo() ;
        record.setImgId(1);
        record.setUploadUserId("A123");
        record.setImgTitle("博文图片");
        record.setSystemType(1) ;
        record.setImgType(2);
        record.setImgUrl("https://avatars0.githubusercontent.com/u/50793885?s=460&amp;v=4");
        record.setLinkUrl("https://avatars0.githubusercontent.com/u/50793885?s=460&amp;v=4");
        record.setShowState(1);
        record.setCreateDate(new Date());
        record.setUpdateDate(record.getCreateDate());
        record.setRemark("知了");
        record.setbEnable("1");
        imgInfoRepository.saveImg(record);
    }
    @Test
    public void test2 (){
        ImgInfo imgInfo = imgInfoRepository.findByImgTitle("博文图片") ;
        System.out.println("imgInfo === &gt;&gt; " + imgInfo);
    }
    @Test
    public void test3 (){
        ImgInfo record = new ImgInfo() ;
        record.setImgId(1);
        record.setUploadUserId("A123");
        record.setImgTitle("知了图片");
        record.setSystemType(1) ;
        record.setImgType(2);
        record.setImgUrl("https://avatars0.githubusercontent.com/u/50793885?s=460&amp;v=4");
        record.setLinkUrl("https://avatars0.githubusercontent.com/u/50793885?s=460&amp;v=4");
        record.setShowState(1);
        record.setCreateDate(new Date());
        record.setUpdateDate(record.getCreateDate());
        record.setRemark("知了");
        record.setbEnable("1");
        long result = imgInfoRepository.updateImgInfo(record) ;
        System.out.println("result == &gt;&gt; " + result);
    }
    @Test
    public void test4 (){
        imgInfoRepository.deleteById(1);
    }
}
```

**参考源码** ：https://gitee.com/cicadasmile/spring-boot-base/tree/master/node15-boot-mongodb