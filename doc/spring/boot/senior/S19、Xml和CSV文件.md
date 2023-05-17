# 一、文档类型简介

## 1、XML文档

XML是可扩展标记语言，是一种用于标记电子文件使其具有结构性的标记语言。标记指计算机所能理解的信息符号，通过此种标记，计算机之间可以处理包含各种的信息比如数据结构，格式等。它可以用来标记数据、定义数据类型，是一种允许用户对自己的标记语言进行定义的源语言。适合网络传输，提供统一的方法来描述和交换应用程序的结构化数据。

## 2、CSV文档

CSV文档，以逗号分隔文档内容值，其文件以纯文本形式存储结构数据。CSV文件由任意数目的记录组成，记录间以某种换行符分隔；每条记录由字段组成，字段间的分隔符是其它字符或字符串，最常见的是逗号。CSV是一种通用的、相对简单的文件格式，通常被用在大数据领域，进行大规模的数据搬运操作。

# 二、XML文件管理

## 1、Dom4j依赖

Dom4j是基于Java编写的XML文件操作的API包，用来读写XML文件。具有性能优异、功能强大和简单易使用的特点。

```xml
<dependency>
    <groupId>dom4j</groupId>
    <artifactId>dom4j</artifactId>
    <version>1.6.1</version>
</dependency>
<dependency>
    <groupId>jaxen</groupId>
    <artifactId>jaxen</artifactId>
    <version>1.1.6</version>
</dependency>
```

## 2、基于API封装方法

涉及对XML文件读取、加载、遍历、创建、修改、删除等常用方法。

```java
public class XmlUtil {
    /**
     * 创建文档
     */
    public static Document getDocument (String filename) {
        File xmlFile = new File(filename) ;
        Document document = null;
        if (xmlFile.exists()){
            try{
                SAXReader saxReader = new SAXReader();
                document = saxReader.read(xmlFile);
            } catch (Exception e){
                e.printStackTrace();
            }
        }
        return document ;
    }

    /**
     * 遍历根节点
     */
    public static Document iteratorNode (String filename) {
        Document document = getDocument(filename) ;
        if (document != null) {
            Element root = document.getRootElement();
            Iterator iterator = root.elementIterator() ;
            while (iterator.hasNext()) {
                Element element = (Element) iterator.next();
                System.out.println(element.getName());
            }
        }
        return document ;
    }

    /**
     * 创建XML文档
     */
    public static void createXML (String filePath) throws Exception {
        // 创建 Document 对象
        Document document = DocumentHelper.createDocument();
        // 创建节点,首个节点默认为根节点
        Element rootElement = document.addElement("project");
        Element parentElement = rootElement.addElement("parent");
        parentElement.addComment("版本描述") ;
        Element groupIdElement = parentElement.addElement("groupId") ;
        Element artifactIdElement = parentElement.addElement("artifactId") ;
        Element versionElement = parentElement.addElement("version") ;
        groupIdElement.setText("SpringBoot2");
        artifactIdElement.setText("spring-boot-starters");
        versionElement.setText("2.1.3.RELEASE");
        //设置输出编码
        OutputFormat format = OutputFormat.createPrettyPrint();
        File xmlFile = new File(filePath);
        format.setEncoding("UTF-8");
        XMLWriter writer = new XMLWriter(new FileOutputStream(xmlFile),format);
        writer.write(document);
        writer.close();
    }

    /**
     * 更新节点
     */
    public static void updateXML (String filePath) throws Exception {
        Document document = getDocument (filePath) ;
        if (document != null){
            // 修改指定节点
            List elementList = document.selectNodes("/project/parent/groupId");
            Iterator iterator = elementList.iterator() ;
            while (iterator.hasNext()){
                Element element = (Element) iterator.next() ;
                element.setText("spring-boot-2");
            }
            //设置输出编码
            OutputFormat format = OutputFormat.createPrettyPrint();
            File xmlFile = new File(filePath);
            format.setEncoding("UTF-8");
            XMLWriter writer = new XMLWriter(new FileOutputStream(xmlFile),format);
            writer.write(document);
            writer.close();
        }
    }

    /**
     * 删除节点
     */
    public static void removeElement (String filePath) throws Exception {
        Document document = getDocument (filePath) ;
        if (document != null){
            // 修改指定节点
            List elementList = document.selectNodes("/project/parent");
            Iterator iterator = elementList.iterator() ;
            while (iterator.hasNext()){
                Element parentElement = (Element) iterator.next() ;
                Iterator parentIterator = parentElement.elementIterator() ;
                while (parentIterator.hasNext()){
                    Element childElement = (Element)parentIterator.next() ;
                    if (childElement.getName().equals("version")) {
                        parentElement.remove(childElement) ;
                    }
                }
            }
            //设置输出编码
            OutputFormat format = OutputFormat.createPrettyPrint();
            File xmlFile = new File(filePath);
            format.setEncoding("UTF-8");
            XMLWriter writer = new XMLWriter(new FileOutputStream(xmlFile),format);
            writer.write(document);
            writer.close();
        }
    }
    public static void main(String[] args) throws Exception {
        String filePath = "F:\\file-type\\project-cf.xml" ;
        // 1、创建文档
        Document document = getDocument(filePath) ;
        System.out.println(document.getRootElement().getName());
        // 2、根节点遍历
        iteratorNode(filePath);
        // 3、创建XML文件
        String newFile = "F:\\file-type\\project-cf-new.xml" ;
        createXML(newFile) ;
        // 4、更新XML文件
        updateXML(newFile) ;
        // 5、删除节点
        removeElement(newFile) ;
    }
}
```

## 3、执行效果图

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/151929_1af27683_5064118.png "19-1.png")

# 三、CSV文件管理

## 1、CSV文件样式

![输入图片说明](https://images.gitee.com/uploads/images/2022/0130/151941_c3f39aca_5064118.png "19-2.png")

这里不需要依赖特定的Jar包，按照普通的文件读取即可。

## 2、文件读取

```java
@Async
@Override
public void readNotify(String path, Integer columnSize) throws Exception {
    File file = new File(path) ;
    String fileName = file.getName() ;
    int lineNum = 0 ;
    if (fileName.startsWith("data-")) {
        InputStreamReader isr = new InputStreamReader(new FileInputStream(file),"GBK") ;
        BufferedReader reader = new BufferedReader(isr);
        List<DataInfo> dataInfoList = new ArrayList<>(4);
        String line  ;
        while ((line = reader.readLine()) != null) {
            lineNum ++ ;
            String[] dataArray = line.split(",");
            if (dataArray.length == columnSize) {
                String cityName = new String(dataArray[1].getBytes(),"UTF-8") ;
                dataInfoList.add(new DataInfo(Integer.parseInt(dataArray[0]),cityName,dataArray[2])) ;
            }
            if (dataInfoList.size() >= 4){
                LOGGER.info("容器数据："+dataInfoList);
                dataInfoList.clear();
            }
        }
        if (dataInfoList.size()>0){
            LOGGER.info("最后数据："+dataInfoList);
        }
        reader.close();
    }
    LOGGER.info("读取数据条数："+lineNum);
}
```

## 3、文件创建

```java
@Async
@Override
public void createCsv(List<String> dataList,String path) throws Exception {
    File file = new File(path) ;
    boolean createFile = false ;
    if (file.exists()){
        boolean deleteFile = file.delete() ;
        LOGGER.info("deleteFile："+deleteFile);
    }
    createFile = file.createNewFile() ;
    OutputStreamWriter ost = new OutputStreamWriter(new FileOutputStream(path),"UTF-8") ;
    BufferedWriter out = new BufferedWriter(ost);
    if (createFile){
        for (String line:dataList){
            if (!StringUtils.isEmpty(line)){
                out.write(line);
                out.newLine();
            }
        }
    }
    out.close();
}
```

## 4、编写测试接口

这里基于Swagger2管理接口测试 。

```java
@Api("Csv接口管理")
@RestController
public class CsvWeb {
    @Resource
    private CsvService csvService ;
    @ApiOperation(value="文件读取")
    @GetMapping("/csv/readNotify")
    public String readNotify (@RequestParam("path") String path,
                              @RequestParam("column") Integer columnSize) throws Exception {
        csvService.readNotify(path,columnSize);
        return "success" ;
    }
    @ApiOperation(value="创建文件")
    @GetMapping("/csv/createCsv")
    public String createCsv (@RequestParam("path") String path) throws Exception {
        List<String> dataList = new ArrayList<>() ;
        dataList.add("1,北京,beijing") ;
        dataList.add("2,上海,shanghai") ;
        dataList.add("3,苏州,suzhou") ;
        csvService.createCsv(dataList,path);
        return "success" ;
    }
}
```

**参考源码**：https://gitee.com/cicadasmile/middle-ware-parent/tree/master/ware18-file-parent