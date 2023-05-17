# 一、SpringMvc简介

## 1、Mvc设计理念

MVC是一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个组件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑，MVC分层有助于管理和架构复杂的应用程序

- M:代表模型`Model`

模型就是数据，应用程序的核心。

- V:代表视图`View`

回显数据的界面,例如JSP就是用来展示模型中的数据。

- C:代表控制器`Controller`

控制器的作用就是根据入参，把不同的响应数据(`Model`)，显示在不同的视图(`View`)上。

## 2、SpringMvc简介

- 框架描述

`SpringMVC`是一种基于`Java`实现的`MVC`设计模式的请求驱动类型的轻量级`Web`框架，出自`Spring`框架全家桶，与`Spring`框架无缝整合，使用了`MVC`架构模式的思想，将`Web`层进行职责解耦。

- 框架优点

结构松散，几乎可以在`SpringMVC`中使用各类视图，各个模块分离而且耦合度非常低，且易于扩展。与`Spring`无缝集成，且简单，灵活，容易上手。

# 二、SpringMvc执行流程

## 1、流程图解

![](https://images.gitee.com/uploads/images/2022/0123/231124_4a887aac_5064118.png "06-1.png")

## 2、步骤描述

(1)、发起请求到前端控制器`DispatcherServlet`;

(2)、前端控制器请求`HandlerMapping`查找,`Handler`可以根据`xml`配置、注解进行查找;

(3)、处理器映射器`HandlerMapping`向前端控制器返回`Handler`;

(4)、前端控制器调用处理器适配器去执行`Handler`;

(5)、处理器适配器去执行`Handler` ;

(6)、`Handler`执行完成给适配器返回`ModelAndView` ;

(7)、处理器适配器向前端控制器返回`ModelAndView`,`ModelAndView`是`springmvc`框架的一个底层对象,包括`Model`和`view `;

(8)、前端控制器请求视图解析器去进行视图解析，根据逻辑视图名解析成真正的视图 ;

(9)、视图解析器向前端控制器返回`View` ;

(10)、前端控制器进行视图渲染,视图渲染将模型数据(在`ModelAndView`对象中)填充到`request`域中;

(11)、前端控制器向用户响应结果 ;

## 3、核心组件

- 前端控制器

`DispatcherServlet`:请求离开浏览器后，最先到达的就是DispatcherServlet，是整个流程控制的中心，作用接收请求，响应结果，相当于转发器，中央处理器。减少各个组件之间的耦合度。

- 处理器映射器

`HandlerMapping`:根据请求的url路由到指定接口，用户请求找到Handler处理器，springmvc提供不同类型映射器，例如：Xml配置方式，注解方式等。

- 处理器适配器

`HandlerAdapter`:按照特定规则去执行Handler，SpringMvc支持多种处理器，各种处理器中的处理方法各不相同，为了解决适应多种处理器，就出现了处理器适配器。

- 处理器

`Handler`：处理用户请求，涉及具体业务逻辑，需要程序员根据业务需求开发。编写Handler时按照HandlerAdapter的规则开发，这样适配器才可以正确执行Handler。

- 视图解析器

`ViewResolver`：负责将请求的响应结果生成View，根据逻辑视图名解析成物理视图名，就是具体页面地址，生成View视图对象，对View进行渲染，通过页面展示给用户。

- 视图

`View`：SpringMvc框架提供很多的View视图类型的支持，包括：jsp、freemarker、pdf等。通过页面标签或页面模版解析模型数据回显到页面，需要根据业务开发具体页面。

# 三、Spring框架整合

## 1、spring-mvc配置

```xml
<!-- 扫描文件 -->
<context:component-scan base-package="com.spring.mvc.controller" />
<!-- MVC默认的注解映射的方式 -->
<mvc:annotation-driven />
<mvc:default-servlet-handler/>
<!-- 视图解析器 -->
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/page/" />
    <property name="suffix" value=".jsp" />
</bean>
```

## 2、Web.xml配置

```xml
<servlet>
    <servlet-name>spring-mvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>spring-mvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

## 3、测试接口

```java
@Controller
public class HelloController {
    @RequestMapping("/getInfo")
    public @ResponseBody String getInfo (String name){
        return name ;
    }
}
```

## 4、常用注解说明

- `@Controller`

标记一个类是Handler，也就是开发的Controller，然后使用@RequestMapping或其他相关注解（@GetMapping、@PostMapping、@PutMapping、@DeleteMapping），用来关联请求和Controller方法之间的映射关系，这样的Controller 就可以被请求访问。

- `@RequestMapping`

处理请求地址映射的注解，可作用于类或方法上。用于类上，表示类中的所有响应请求的方法都是以类上标注地址作为父路径。

- `@requestParam`

主要用于在SpringMvc框架的控制层获取参数，三个常用参数：defaultValue表示设置默认值，required 通过boolean设置是否是必须要传入的参数，value值表示传入的参数名称。

- `@RequestBody`

接收请求体中传递给后端的Json字符串数据的，GET方式无请求体，所以使用@RequestBody接收数据时，不能使用GET方式提交数据，需要用POST方式进行提交。

- `@ResponseBody`

该注解用于方法的返回对象，可以通过配置转换器为指定数据响应格式，如果希望返回的数据不是View试图页面，而是指定数据格式的时候使用，例如：Json、Xml等。

- `@Autowired`

按照类型（byType）装配依赖对象，默认情况下它要求依赖对象必须存在，如果允许null值，可以设置它的required属性为false。如果想使用按照名称（byName）来装配，可以结合@Qualifier注解一起使用。

- `@Resource`

按照ByName自动注入，需要导入包javax.annotation.Resource。@Resource有两个重要的属性：name和type，而Spring将@Resource注解的name属性解析为bean的名字，而type属性则解析为bean的类型。

- `@PathVariable`

用于将请求URL中的模板变量映射到功能处理方法的参数上，即取出uri模板中的变量作为参数。

# 四、常见参数映射

## 1、普通映射

```java
@RequestMapping("/getSum")
public Integer getSum (int a,int b){
    return a+b ;
}
```

测试：

`http://localhost:6003/getSum?a=1&b=2`

传参名称和方法参数保持一致。

## 2、指定参数名

```java
@RequestMapping("/getInfo")
public String getInfo (@RequestParam("name") String var1,
                       @RequestParam("say") String var2){
    return var1+":"+var2 ;
}
```

测试：

`http://localhost:6003/getInfo?name=cica&say=hello`

传参名和 @RequestParam 指定的参数名要对应。

## 3、数组参数

```java
@GetMapping("/getArray")
public String getArray (String[] ids){
    return ids[0]+"-"+ids[1] ;
}
```
测试：

`http://localhost:6003/getArray?ids=2&ids=3`

传递并解析数组类型的参数格式。

## 4、Map参数

```java
@RequestMapping("/getMap")
public String getMap (@RequestParam Map<String,String> paramMap){
    return paramMap.get("name") ;
}
```

测试：

`http://localhost:6003/getCityEntity?province=浙江&name=杭州`

这里以Post方式将相关参数传递CityEntity实体对象中。

## 5、包装参数

```java
@PostMapping("/getCityEntity")
public CityEntity getCityEntity (CityEntity cityEntity){
    return cityEntity ;
}
```

测试：

`http://localhost:6003/getCityEntity?province=浙江&name=杭州`

这里以Post方式将相关参数传递CityEntity实体对象中。

## 6、Rest风格参数

```java
@GetMapping("/getId/{id}")
public String getId (@PathVariable("id") Integer id){
    return "id="+id ;
}
```

测试：

`http://localhost:6003/getId/1`

RestFul 风格参数映射。

# 五、Mvc工程代码分层

MVC模式与代码分层策略，MVC全名是ModelViewController即模型－视图－控制器，作为一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个部件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑，这是一种开发模式，但并不是实际开发中代码的分层模式，通常SSM框架的后端代码分层如下：

![](https://oscimg.oschina.net/oscnet/up-17ebf8ba71bbb1cd3c30510d5c6cb28a7a9.png)

- controller控制层：定义服务端接口，入参出参，和一些入参校验；
- service业务服务层：组装业务逻辑，业务校验，构建控制层需要的参数模型；
- dao数据交互层：提供服务层需要的数据查询方法，处理数据交互条件相关的逻辑；
- mapper持久层：基于mybatis框架需要的原生支持，目前很常用的持久层组件；

**源码参考：** https://gitee.com/cicadasmile/spring-mvc-parent/tree/master/spring-base-node05