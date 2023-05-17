# 一、逆向工程简介

在Java开发中，持久层最常用的框架就是mybatis，该框架需要编写sql语句，mybatis官方提供逆向工程，可以把数据表自动生成执行所需要的基础代码，例如：mapper接口，sql映射文件，pojo实体类等，避免基础代码维护的繁杂过程。

在实际的使用中，常用的逆向工程方式如上，mybatis框架，mybatis-plus框架，插件方式。

# 二、Mybatis方式

## 1、基础描述

基于xml配置的方式，生成mybatis基础代码，包括mapper接口，Mapper映射文件，pojo实体类，PojoExample条件工具类。

## 2、配置文件

注意这里的targetProject需要配置自定义路径位置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
		PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
		"http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
	<context id="testTables" targetRuntime="MyBatis3">
		<commentGenerator>
			<!-- 是否去除自动生成的注释 true：是 ： false:否 -->
			<property name="suppressAllComments" value="true"/>
			<property name="suppressDate" value="false"/>
			<!-- 是否添加数据表中字段的注释 true：是 ： false:否 -->
			<property name="addRemarkComments" value="true"/>
		</commentGenerator>

		<!--数据库的信息：驱动类、连接地址、用户名、密码 -->
		<jdbcConnection driverClass="com.mysql.jdbc.Driver"
			connectionURL="jdbc:mysql://localhost:3306/defined-log?tinyInt1isBit=false"
			userId="root" password="123456">
		</jdbcConnection>

		<!--
			默认false，把JDBC decimal 和 numeric 类型解析为 Integer
		    设置true时把JDBC decimal 和 numeric 类型解析为BigDecimal
		-->
		<javaTypeResolver>
			<property name="forceBigDecimals" value="false" />
		</javaTypeResolver>

		<!-- 生成POJO类的位置 -->
		<javaModelGenerator targetPackage="com.generator.mybatis.pojo"
			targetProject="存放路径">
			<property name="enableSubPackages" value="true" />
			<property name="trimStrings" value="true" />
		</javaModelGenerator>

		<!-- 生成Mapper映射文件的位置 -->
		<sqlMapGenerator targetPackage="com.generator.mybatis.xml"
			targetProject="存放路径">
			<property name="enableSubPackages" value="true" />
		</sqlMapGenerator>

		<!-- 生成Mapper接口的位置 -->
		<javaClientGenerator type="XMLMAPPER" targetPackage="com.generator.mybatis.mapper"
			targetProject="存放路径">
			<property name="enableSubPackages" value="true" />
		</javaClientGenerator>

		<!-- 指定数据库表 -->
		<table schema="" tableName="dt_defined_log" domainObjectName="DefinedLog"/>

	</context>
</generatorConfiguration>
```

## 3、启动类

读取配置文件，并执行。

```java
public class GeneratorMybatis {

    public void generator() throws Exception {
        List<String> warnings = new ArrayList<String>();
        boolean overwrite = true;
        File configFile = Resources.getResourceAsFile("generatorConfig.xml");
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(configFile);
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);
        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config,callback, warnings);
        myBatisGenerator.generate(null);
    }

    public static void main(String[] args) throws Exception {
        try {
            GeneratorMybatis generatorMybatis = new GeneratorMybatis();
            generatorMybatis.generator();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# 三、MybatisPlus方式

## 1、基础描述

MybatisPlus相比Mybatis提供更多增强的能力，单表操作基本都封装好了，所以生成的mapper映射文件简洁很多，需要注意ServiceImpl关键类和BaseMapper接口。

## 2、核心启动类

这里的配置可以基于很多自定义的策略，案例生成的代码已经传到仓库，可以自行下载查看。

```java
public class GeneratorMybatisPlus {

    public static void main(String[] args) {
        // 代码生成器
        AutoGenerator autoGenerator = new AutoGenerator();
        // 全局配置
        GlobalConfig globalConfig = new GlobalConfig();
        //生成文件的输出目录
        String path="存放路径";
        globalConfig.setOutputDir(path);
        // Author设置作者
        globalConfig.setAuthor("mybatis-plus");
        // 文件覆盖
        globalConfig.setFileOverride(true);
        // 生成后打开文件
        globalConfig.setOpen(false);
        // 自定义文件名风格，%s自动填充表实体属性
        globalConfig.setMapperName("%sMapper");
        globalConfig.setXmlName("%sMapper");
        globalConfig.setServiceName("%sDao");
        globalConfig.setServiceImplName("%sDaoImpl");
        globalConfig.setEntityName("%s");
        globalConfig.setControllerName("%sController");
        autoGenerator.setGlobalConfig(globalConfig);

        // 数据源配置
        DataSourceConfig dataSourceConfig = new DataSourceConfig();
        dataSourceConfig.setDbType(DbType.MYSQL);
        dataSourceConfig.setTypeConvert(new MySqlTypeConvert());
        dataSourceConfig.setUrl("jdbc:mysql://localhost:3306/defined-log?tinyInt1isBit=false");
        dataSourceConfig.setDriverName("com.mysql.jdbc.Driver");
        dataSourceConfig.setUsername("root");
        dataSourceConfig.setPassword("123456");
        autoGenerator.setDataSource(dataSourceConfig);

        // 包名配置
        PackageConfig packageConfig = new PackageConfig();
        // 父包和子包名分开处理
        packageConfig.setParent("com.generator.mybatis.plus");
        packageConfig.setController("web");
        packageConfig.setEntity("pojo");
        packageConfig.setMapper("mapper");
        packageConfig.setService("dao");
        packageConfig.setServiceImpl("dao.impl");
        autoGenerator.setPackageInfo(packageConfig);

        // 生成策略配置
        StrategyConfig strategy = new StrategyConfig();
        //设置命名格式
        strategy.setNaming(NamingStrategy.underline_to_camel);
        strategy.setColumnNaming(NamingStrategy.underline_to_camel);
        // 实体是否为lombok模型，默认 false
        strategy.setEntityLombokModel(true);
        //生成 @RestController 控制器
        strategy.setRestControllerStyle(true);
        // 驼峰转连字符
        strategy.setControllerMappingHyphenStyle(true);
        //表和前缀处理
        strategy.setInclude("dt_defined_log".split(","));
        String[] tablePre = new String[]{"dt_"};
        strategy.setTablePrefix(tablePre);
        autoGenerator.setStrategy(strategy);
        // 执行，以上相关参数可以基于动态输入获取
        autoGenerator.execute();
    }
}
```

该方式是当前mybatis框架最流行的开发方式，代码会简洁很多。

# 四、插件工具

## 1、配置数据库

这里选择MySQL数据源，后续根据提示需要下载驱动配置。

![](https://images.gitee.com/uploads/images/2022/0212/202706_dd41e974_5064118.png "01-2.png")

## 2、连接配置

![](https://images.gitee.com/uploads/images/2022/0212/202730_91332f98_5064118.png "01-3.png")

Url地址，账号，密码，获取连接。

## 3、插件使用

这里选择的是安装EasyCode插件。

![](https://images.gitee.com/uploads/images/2022/0212/202752_a7daa8e3_5064118.png "01-4.png")

根据配置，生成逆向工程文件，整体思路和上述两种方式一致。

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap04-format-tool/case01-generator-code