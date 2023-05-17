# 一、页面静态化

## 1、动静态页面

**静态页面**

即静态网页，指已经装载好内容HTML页面，无需经过请求服务器数据和编译过程，直接加载到客户浏览器上显示出来。通俗的说就是生成独立的HTML页面，且不与服务器进行数据交互。

优缺点描述：

- 静态网页的内容稳定，页面加载速度极快；
- 不与服务器交互，提升安全性；
- 静态网页的交互性差，数据实时性很低；
- 维度成本高，生成很多HTML页面；

**动态页面**

指跟静态网页相对的一种网页编程技术，页面的内容需要请求服务器获取，在不考虑缓存的情况下，服务接口的数据变化，页面加载的内容也会实时变化，显示的内容却是随着数据库操作的结果而动态改变的。

优缺点描述：

- 动态网页的实时获取数据，延迟性低；
- 依赖数据库交互，页面维护成本很低；
- 与数据库实时交互，安全控制的成本高；
- 页面加载速度十分依赖数据库和服务的性能；

动态页面和静态页面有很强的相对性，对比之下也比较好理解。

## 2、应用场景

动态页面静态化处理的应用场景非常多，例如：

- 大型网站的头部和底部，静态化之后统一加载；
- 媒体网站，内容经过渲染，直接转为HTML网页；
- 高并发下，CDN边缘节点代理的静态网页；
- 电商网站中，复杂的产品详情页处理；

静态化技术的根本：提示服务的响应速度，或者说使响应节点提前，如一般的流程，页面(客户端)请求服务，服务处理，响应数据，页面装载，一系列流程走下来不仅复杂，而且耗时，如果基于静态化技术处理之后，直接加载静态页面，好了请求结束。

# 二、流程分析

静态页面转换是一个相对复杂的过程，其中核心流程如下：

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/152142_94303796_5064118.png "23-1.png")

- 开发一个页面模板，即静态网页样式；
- 提供接口，给页面模板获取数据；
- 页面模板中编写数据接口返参的解析流程；
- 基于解析引擎，把数据和页面模板合并；
- 页面模板内容加载完成后转换为HTML静态页面；
- HTML静态页面上传到文件服务器；
- 客户端(Client)获取静态页面的url加载显示；

主流程大致如上，如果数据接口响应参数有变，则需要重新生成静态页，所以在数据的加载实时性上面会低很多。

# 三、代码实现案例

## 1、基础依赖

FreeMarker是一款模板引擎：即一种基于模板和要改变的数据，并用来生成输出文本（HTML网页、电子邮件、配置文件、源代码等）的通用工具。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

## 2、页面模板

这里既使用FreeMarker开发的模板样式。

```html
<html>
<head>
    <title>PageStatic</title>
</head>
<body>
主题：${myTitle}<br/>
<#assign text="{'auth':'cicada','date':'2020-07-16'}" />
<#assign data=text?eval />
作者：${data.auth} 日期：${data.date}<br/>
<table class="table table-striped table-hover table-bordered" id="editable-sample">
    <thead>
    <tr>
        <th>规格描述</th>
        <th>产品详情</th>
    </tr>
    </thead>
    <tbody>
             <#list tableList as info>
             <tr class="">
                 <td>${info.desc}</td>
                 <td><img src="${info.imgUrl}" height="80" width="80"></td>
             </tr>
             </#list>
    </tbody>
</table><br/>
<#list imgList as imgIF>
    <img src="${imgIF}" height="300" width="500">
</#list>
</body>
</html>
```

FreeMarker的语法和原有的HTML语法基本一致，但是具有一套自己的数据处理标签，用起来不算复杂。

## 3、解析过程

通过解析，把页面模板和数据接口的数据合并到一起即可。

```java
@Service
public class PageServiceImpl implements PageService {

    private static final Logger LOGGER = LoggerFactory.getLogger(PageServiceImpl.class) ;
    private static final String PATH = "/templates/" ;

    @Override
    public void ftlToHtml() throws Exception {
        // 创建配置类
        Configuration configuration = new Configuration(Configuration.getVersion());
        // 设置模板路径
        String classpath = this.getClass().getResource("/").getPath();
        configuration.setDirectoryForTemplateLoading(new File(classpath + PATH));
        // 加载模板
        Template template = configuration.getTemplate("my-page.ftl");
        // 数据模型
        Map<String, Object> map = new HashMap<>();
        map.put("myTitle", "页面静态化(PageStatic)");
        map.put("tableList",getList()) ;
        map.put("imgList",getImgList()) ;
        // 静态化页面内容
        String content = FreeMarkerTemplateUtils.processTemplateIntoString(template, map);
        LOGGER.info("content:{}",content);
        InputStream inputStream = IOUtils.toInputStream(content,"UTF-8");
        // 输出文件
        FileOutputStream fileOutputStream = new FileOutputStream(new File("F:/page/newPage.html"));
        IOUtils.copy(inputStream, fileOutputStream);
        // 关闭流
        inputStream.close();
        fileOutputStream.close();
    }

    private List<TableInfo> getList (){
        List<TableInfo> tableInfoList = new ArrayList<>() ;
        tableInfoList.add(new TableInfo(Constant.desc1, Constant.img01));
        tableInfoList.add(new TableInfo(Constant.desc2,Constant.img02));
        return tableInfoList ;
    }

    private List<String> getImgList (){
        List<String> imgList = new ArrayList<>() ;
        imgList.add(Constant.img02) ;
        imgList.add(Constant.img02) ;
        return imgList ;
    }
}
```

生成后的HTML页面直接使用浏览器打开即可，不再需要依赖任何数据接口服务。

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware22-page-static