# 一、Log4j2日志简介

日志打印是了解Web项目运行的最直接方式，所以在项目开发中是需要首先搭建好的环境。

## 1、Log4j2特点

- 核心特点

相比与其他的日志系统，log4j2丢数据这种情况少；disruptor技术，在多线程环境下，性能高；并发的特性，减少了死锁的发生。

- 性能测试

![](https://images.gitee.com/uploads/images/2021/0829/142254_e7ac1c71_5064118.png "02-1.png")

![](https://images.gitee.com/uploads/images/2021/0829/142310_f3d8e8ca_5064118.png "02-2.png")

## 2、日志打印之外观模式

每一种日志框架都有自己单独的API，要使用对应的框架就要使用其对应的API，增加应用程序代码和日志框架的耦合性。
《阿里巴巴Java开发手册》，其中有一条规范做了『强制』要求：

![](https://images.gitee.com/uploads/images/2021/0829/142343_c0596951_5064118.png "02-3.png")

- SLF4J￼

Java简易日志门面（Simple Logging Facade for Java，缩写SLF4J），是一套包装Logging 框架的界面程式，以外观模式实现。

# 二、配置日志打印

## 1、项目结构

![](https://images.gitee.com/uploads/images/2021/0829/142413_d5a0ef4e_5064118.png "02-4.png")

## 2、不同环境的日志配置

使用最直接的方式，不同环境加载不同的日志配置。

- 开发环境配置

```
logging:
  config: classpath:log4j2-boot-dev.xml
```

- 生产环境配置

```
logging:
  config: classpath:log4j2-boot-pro.xml
```

## 3、Log4j2的配置文件

```xml
<!--?xml version="1.0" encoding="UTF-8"?-->
<!--monitorInterval：Log4j2 自动检测修改配置文件和重新配置本身，设置间隔秒数-->
<configuration monitorinterval="5">
    <!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
    <!--变量配置-->
    <properties>
        <!-- 格式化输出：
            %date表示日期，%thread表示线程名，
            %-5level：级别从左显示5个字符宽度
            %msg：日志消息，%n是换行符-->
        <!-- %logger{36} 表示 Logger 名字最长36个字符 -->
        <property name="LOG_PATTERN" value="%date{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n" />
        <!-- 定义日志存储的路径，不要配置相对路径 -->
        <property name="FILE_PATH" value="E:/logs/dev" />
        <property name="FILE_NAME" value="boot-log4j2" />
    </properties>

    <appenders>
        <console name="Console" target="SYSTEM_OUT">
            <!--输出日志的格式-->
            <patternlayout pattern="${LOG_PATTERN}" />
            <!--控制台只输出level及其以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <thresholdfilter level="info" onMatch="ACCEPT" onMismatch="DENY" />
        </console>

        <!--文件会打印出所有信息-->
        <file name="Filelog" filename="${FILE_PATH}/log4j2.log" append="true">
            <patternlayout pattern="${LOG_PATTERN}" />
        </file>

        <!-- 这个会打印出所有的info及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <rollingfile name="RollingFileInfo" filename="${FILE_PATH}/info.log" filepattern="${FILE_PATH}/${FILE_NAME}-INFO-%d{yyyy-MM-dd}_%i.log.gz">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <thresholdfilter level="info" onMatch="ACCEPT" onMismatch="DENY" />
            <patternlayout pattern="${LOG_PATTERN}" />
            <policies>
                <!--interval属性用来指定多久滚动一次，默认是1 hour-->
                <timebasedtriggeringpolicy interval="1" />
                <sizebasedtriggeringpolicy size="10MB" />
            </policies>
            <!-- DefaultRolloverStrategy同一文件夹下15个文件开始覆盖-->
            <defaultrolloverstrategy max="15" />
        </rollingfile>

        <!-- 这个会打印出所有的warn及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <rollingfile name="RollingFileWarn" filename="${FILE_PATH}/warn.log" filepattern="${FILE_PATH}/${FILE_NAME}-WARN-%d{yyyy-MM-dd}_%i.log.gz">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <thresholdfilter level="warn" onMatch="ACCEPT" onMismatch="DENY" />
            <patternlayout pattern="${LOG_PATTERN}" />
            <policies>
                <!--interval属性用来指定多久滚动一次，默认是1 hour-->
                <timebasedtriggeringpolicy interval="1" />
                <sizebasedtriggeringpolicy size="10MB" />
            </policies>
            <!-- DefaultRolloverStrategy同一文件夹下15个文件开始覆盖-->
            <defaultrolloverstrategy max="15" />
        </rollingfile>

        <!-- 这个会打印出所有的error及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <rollingfile name="RollingFileError" filename="${FILE_PATH}/error.log" filepattern="${FILE_PATH}/${FILE_NAME}-ERROR-%d{yyyy-MM-dd}_%i.log.gz">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <thresholdfilter level="error" onMatch="ACCEPT" onMismatch="DENY" />
            <patternlayout pattern="${LOG_PATTERN}" />
            <policies>
                <!--interval属性用来指定多久滚动一次，默认是1 hour-->
                <timebasedtriggeringpolicy interval="1" />
                <sizebasedtriggeringpolicy size="10MB" />
            </policies>
            <!-- DefaultRolloverStrategy同一文件夹下15个文件开始覆盖-->
            <defaultrolloverstrategy max="15" />
        </rollingfile>
    </appenders>

    <!--Logger节点用来单独指定日志的形式，比如要为指定包下的class指定不同的日志级别等。-->
    <!--然后定义loggers，只有定义了logger并引入的appender，appender才会生效-->
    <loggers>
        <!--过滤掉spring和mybatis的一些无用的DEBUG信息-->
        <logger name="org.mybatis" level="info" additivity="false">
            <appenderref ref="Console" />
        </logger>
        <!--监控系统信息-->
        <!--若是additivity设为false，则 子Logger 只会在自己的appender里输出，而不会在 父Logger 的appender里输出。-->
        <logger name="org.springframework" level="info" additivity="false">
            <appenderref ref="Console" />
        </logger>
        <root level="info">
            <appender-ref ref="Console" />
            <appender-ref ref="Filelog" />
            <appender-ref ref="RollingFileInfo" />
            <appender-ref ref="RollingFileWarn" />
            <appender-ref ref="RollingFileError" />
        </root>
    </loggers>

</configuration>
```

# 三、测试日志打印

## 1、简单的测试程序

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class Log4j2Controller {
    private static final Logger LOGGER = LoggerFactory.getLogger(Log4j2Controller.class);
    /**
     * 日志级别
     * OFF &gt; FATAL &gt; ERROR &gt; WARN &gt; INFO &gt; DEBUG &gt; TRACE &gt; ALL
     */
    @RequestMapping("/printLog")
    public String printLog (){
        LOGGER.error("ERROR 级别日志");
        LOGGER.warn("WARN 级别日志");
        LOGGER.info("INFO 级别日志");
        LOGGER.debug("DEBUG 级别日志");
        LOGGER.trace("TRACE 级别日志");
        return "success" ;
    }
}
```

## 2、测试效果图

![](https://images.gitee.com/uploads/images/2021/0829/142515_72f166e0_5064118.png "02-5.png")

**参考源码** ：https://gitee.com/cicadasmile/spring-boot-base/tree/master/node02-boot-log4j