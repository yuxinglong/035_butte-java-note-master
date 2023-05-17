# 一、文件上传

文件上传是项目开发中一个很常用的功能，常见的如头像上传，各类文档数据上传等。SpringBoot使用MultiPartFile接收来自表单的file文件,然后执行上传文件。该案例基于SpringBoot2.0中yml配置，管理文件上传的常见属性。该案例演示单文件上传和多文件上传。

# 二、搭建文件上传界面

## 1、引入页面模板Jar包

```xml
<dependency>
    <groupid>org.springframework.boot</groupid>
    <artifactid>spring-boot-starter-thymeleaf</artifactid>
</dependency>
```

## 2、编写简单的上传页面

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
<hr/>
<h3>1、单文件上传</h3>
<form method="POST" action="/upload1" enctype="multipart/form-data">
    上传人：<input type="text" name="userName" /><br/>
    文件一：<input type="file" name="file" /><br/>
    <input type="submit" value="Submit" />
</form>
<hr/>
<h3>2、多文件上传</h3>
<form method="POST" action="/upload2" enctype="multipart/form-data">
    上传人：<input type="text" name="userName" /><br/>
    文件一：<input type="file" name="file" /><br/>
    文件二：<input type="file" name="file" /><br/><br/>
    <input type="submit" value="Submit" />
</form>
</body>
</html>
<hr/>
```

## 3、配置页面入口

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
@Controller
public class PageController {
    /**
     * 上传页面
     */
    @GetMapping("/uploadPage")
    public String uploadPage (){
        return "upload" ;
    }
}
```

# 三、SpringBoot2整合

## 1、核心配置文件

- 上传文件单个限制：max-file-size: 5MB
- 上传文件总大小限制：max-request-size: 6MB

```
spring:
  application:
    # 应用名称
    name: node14-boot-file
  servlet:
    multipart:
      # 启用
      enabled: true
      # 上传文件单个限制
      max-file-size: 5MB
      # 总限制
      max-request-size: 6MB
```

## 2、文件上传核心代码

- 如果单个文件大小超出1MB，抛出异常：FileSizeLimitExceededException;
- 如果上传文件总大小超过6MB，抛出异常：SizeLimitExceededException;

这样就完全验证了YML文件中的配置，有效且正确。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;
import javax.servlet.http.HttpServletRequest;
import java.io.File;
import java.util.Map;
@RestController
public class FileController {
    private static final Logger LOGGER = LoggerFactory.getLogger(FileController.class) ;
    /**
     * 测试单个文件上传
     */
    @RequestMapping("/upload1")
    public String upload1 (HttpServletRequest request, @RequestParam("file") MultipartFile file){
        Map<string, string[]> paramMap = request.getParameterMap() ;
        if (!paramMap.isEmpty()){
            LOGGER.info("paramMap == &gt;&gt;{}",paramMap);
        }
        try{
            if (!file.isEmpty()){
                // 打印文件基础信息
                LOGGER.info("Name == &gt;&gt;{}",file.getName());
                LOGGER.info("OriginalFilename == &gt;&gt;{}",file.getOriginalFilename());
                LOGGER.info("ContentType == &gt;&gt;{}",file.getContentType());
                LOGGER.info("Size == &gt;&gt;{}",file.getSize());
                // 文件输出地址
                String filePath = "F:/boot-file/" ;
                new File(filePath).mkdirs();
                File writeFile = new File(filePath, file.getOriginalFilename());
                file.transferTo(writeFile);
            }
            return "success" ;
        } catch (Exception e){
            e.printStackTrace();
            return "系统异常" ;
        }
    }
    /**
     * 测试多文件上传
     */
    @RequestMapping("/upload2")
    public String upload2 (HttpServletRequest request, @RequestParam("file") MultipartFile[] fileList){
        Map<string, string[]> paramMap = request.getParameterMap() ;
        if (!paramMap.isEmpty()){
            LOGGER.info("paramMap == &gt;&gt;{}",paramMap);
        }
        try{
            if (fileList.length &gt; 0){
                for (MultipartFile file:fileList){
                    // 打印文件基础信息
                    LOGGER.info("Name == &gt;&gt;{}",file.getName());
                    LOGGER.info("OriginalFilename == &gt;&gt;{}",file.getOriginalFilename());
                    LOGGER.info("ContentType == &gt;&gt;{}",file.getContentType());
                    LOGGER.info("Size == &gt;&gt;{}",file.getSize());
                    // 文件输出地址
                    String filePath = "F:/boot-file/" ;
                    new File(filePath).mkdirs();
                    File writeFile = new File(filePath, file.getOriginalFilename());
                    file.transferTo(writeFile);
                }
            }
            return "success" ;
        } catch (Exception e){
            return "fail" ;
        }
    }
}
```

**参考源码** ：https://gitee.com/cicadasmile/spring-boot-base/tree/master/node14-boot-file