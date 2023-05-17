# 一、Servlet简介

Java编写的服务器端程序，具有独立于平台和协议的特性，主要功能在于交互式地浏览和生成数据，生成动态Web内容。使用Servlet，可以收集来自网页表单的用户输入，呈现来自数据库或者其他源的记录，还可以动态创建网页。

# 二、实现方式

## 1、继承HttpServlet

- API简介

继承自 GenericServlet. 遵守 HTTP协议实现，以设计模式的角度看，HttpServlet担任抽象模板角色，模板方法：由service()方法担任。基本方法：由doPost()、doGet()等方法担任。service()方法流程，省略了部分判断逻辑。该方法调用七个do方法中的一个或几个，完成对客户端请求的响应。这些do方法需要由HttpServlet的具体子类提供，这种API封装是典型的模板方法模式。

- 代码案例

```
public class ServletOneImpl extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=utf-8");
        response.getWriter().print("执行：doGet");
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=utf-8");
        response.getWriter().print("执行：doPost");
    }
}
```

## 2、继承GenericServlet

- API 简介

Servlet 接口和 ServletConfig 接口的实现类. 一个抽象类. 其中的 service 方法为抽象方法。

- 代码案例

```
public class ServletTwoImpl extends GenericServlet {
    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse)
            throws ServletException, IOException {
        HttpServletResponse response = (HttpServletResponse)servletResponse ;
        response.setContentType("text/html;charset=utf-8");
        response.getWriter().print("执行：service");
    }
}
```

## 3、实现Servlet接口

- API 简介

Servlet是一个接口，其中包含init、getServletConfig、service、getServletInfo、destroy几个核心方法。

- 代码案例

```
public class ServletThreeImpl implements Servlet {
    @Override
    public void init(ServletConfig servletConfig) throws ServletException {
        servletConfig.getServletName();
        System.out.println("init 被调用...");
    }
    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse)
            throws ServletException, IOException {
        System.out.println("ThreadId："+Thread.currentThread().getId());
        System.out.println("service 被调用...");
        HttpServletResponse response = (HttpServletResponse)servletResponse ;
        response.getWriter().print("Servlet.Life");
    }
    @Override
    public void destroy() {
        System.out.println("destroy 被调用...");
    }
    @Override
    public ServletConfig getServletConfig() {
        System.out.println("getServletConfig 被调用...");
        return null;
    }
    @Override
    public String getServletInfo() {
        System.out.println("getServletInfo 被调用...");
        return null;
    }
}
```

# 三、生命周期

- 加载和实例化

当Servlet容器启动或客户端发送请求时，Servlet容器会查找是否存在该Servlet实例，若存在，则直接读取该实例响应请求；如果不存在，就创建一个Servlet实例(属于单例设计模式)。load-on-startup 可以配置创建时序。

- 初始化：init()

实例化后，Servlet容器将调用init方法一次，初始化当前 Servlet。

- 服务：service()

初始化后，Servlet处于响应请求的就绪状态。当接收到客户端请求时，调用service()的方法处理客户端请求，HttpServlet的service()方法会根据不同的请求 调用不同的模板方法。

- 销毁：destroy()

当Servlet容器关闭时，Servlet实例也随时销毁。关闭 Tomcat 服务时可以通过日志打印看到该方法的执行。

# 四、运行配置

## 1、web.xml配置

```
<servlet>
    <servlet-name>servletOneImpl</servlet-name>
    <servlet-class>com.node01.servlet.impl.ServletOneImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletOneImpl</servlet-name>
    <url-pattern>/servletOneImpl</url-pattern>
</servlet-mapping>
<servlet>
    <servlet-name>servletTwoImpl</servlet-name>
    <servlet-class>com.node01.servlet.impl.ServletTwoImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletTwoImpl</servlet-name>
    <url-pattern>/servletTwoImpl</url-pattern>
</servlet-mapping>
<servlet>
    <servlet-name>servletThreeImpl</servlet-name>
    <servlet-class>com.node01.servlet.impl.ServletThreeImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletThreeImpl</servlet-name>
    <url-pattern>/servletThreeImpl</url-pattern>
</servlet-mapping>
```

请求：`http://localhost:6003/servletOneImpl` 测试。

- servlet-name：Servlet 注册名称。
- servlet-class：Servlet 全路径类名。
- serlvet-mapping：同一个Servlet可以被映射到多个URL上。
- url-pattern：Servlet 访问的映射路径。

## 2、线程池运行

观察上述第三种Servlet实现方式的日志打印：`Thread.currentThread().getId());`。
```
ThreadId：32
ThreadId：33
ThreadId：32
ThreadId：31
ThreadId：32
```
这里不难发现，Servlet以线程池的方式执行的。


# 五、核心API简介

## 1、Servlet执行流程

Servlet是JavaWeb的三大组件之一（Servlet、Filter、Listener），它属于动态资源。Servlet的作用是处理请求，服务器会把接收到的请求交给Servlet来处理，在Servlet中通常需要：接收请求数据；处理请求；完成响应。

## 2、核心API简介

API | 作用描述
---|---
ServletConfig | 获取servlet初始化参数和servletContext对象。
ServletContext | 在整个Web应用的动态资源之间共享数据。
ServletRequest | 封装Http请求信息，在请求时创建。
ServletResponse | 封装Http响应信息，在请求时创建。

# 六、ServletConfig接口

## 1、接口简介

容器在初始化servlet时，为该servlet创建一个servletConfig对象，并将这个对象通过init()方法来传递并保存在此Servlet对象中。核心作用：1.获取初始化信息;2.获取ServletContext对象。

## 2、代码案例

- 配置文件

```xml
<servlet>
    <init-param>
        <param-name>my-name</param-name>
        <param-value>cicada</param-value>
    </init-param>
    <servlet-name>servletOneImpl</servlet-name>
    <servlet-class>com.node02.servlet.impl.ServletOneImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletOneImpl</servlet-name>
    <url-pattern>/servletOneImpl</url-pattern>
</servlet-mapping>
```

- API用法

```java
public class ServletOneImpl implements Servlet {

    @Override
    public void init(ServletConfig servletConfig) throws ServletException {
        String servletName = servletConfig.getServletName() ;
        System.out.println("servletName="+servletName);
        String myName = servletConfig.getInitParameter("my-name") ;
        System.out.println("myName="+myName);
        Enumeration paramNames = servletConfig.getInitParameterNames() ;
        while (paramNames.hasMoreElements()){
            String paramKey = String.valueOf(paramNames.nextElement()) ;
            String paramValue = servletConfig.getInitParameter(paramKey) ;
            System.out.println("paramKey="+paramKey+";paramValue="+paramValue);
        }
        ServletContext servletContext = servletConfig.getServletContext() ;
        servletContext.setAttribute("cicada","smile");
    }
}
```

# 七、ServletContext接口

## 1、接口简介

一个项目只有一个ServletContext对象，可以在多个Servlet中来获取这个对象，使用它可以给多个Servlet传递数据，该对象在Tomcat启动时就创建，在Tomcat关闭时才会销毁！作用是在整个Web应用的动态资源之间共享数据。

- 获取方式

```
1、ServletConfig#getServletContext()；
2、GenericServlet#getServletContext();
3、HttpSession#getServletContext()
4、ServletContextEvent#getServletContext()
```

## 2、四大域对象

ServletContext是JavaWeb四大域对象之一：
```
1、PageContext；
2、ServletRequest；
3、HttpSession；
4、ServletContext；
```
所有域对象都有存取数据的功能，因为域对象内部有一个Map，用来存储数据。

## 3、代码案例

- 配置文件

```XML
<context-param>
    <param-name>my-blog</param-name>
    <param-value>2019-11-19</param-value>
</context-param>
<servlet>
    <servlet-name>servletTwoImpl</servlet-name>
    <servlet-class>com.node02.servlet.impl.ServletTwoImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletTwoImpl</servlet-name>
    <url-pattern>/servletTwoImpl</url-pattern>
</servlet-mapping>
```

- API用法

```java
public class ServletTwoImpl extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=utf-8");
        // 1、参数传递
        ServletContext servletContext = this.getServletContext() ;
        String value = String.valueOf(servletContext.getAttribute("cicada")) ;
        System.out.println("value="+value);
        // 2、获取初始化参数
        String myBlog = servletContext.getInitParameter("my-blog") ;
        System.out.println("myBlog="+myBlog);
        // 3、获取应用信息
        String servletContextName = servletContext.getServletContextName() ;
        System.out.println("servletContextName="+servletContextName);
        // 4、获取路径
        String pathOne = servletContext.getRealPath("/") ;
        String pathTwo = servletContext.getRealPath("/WEB-INF/") ;
        System.out.println("pathOne="+pathOne+";pathTwo="+pathTwo);
        response.getWriter().print("执行：doGet; value："+value);
    }
}
```

# 八、ServletRequest接口

## 1、接口简介

HttpServletRequest接口继承ServletRequest接口，用于封装请求信息，该对象在用户每次请求servlet时创建并传入servlet的service()方法，在该方法中，传入的servletRequest将会被强制转化为HttpservletRequest对象来进行HTTP请求信息的处理。核心作用：1.获取请求报文信息;2.获取网络连接信息;3.获取请求域属性信息。

## 2、代码案例

- 配置文件

```xml
<servlet>
    <servlet-name>servletThreeImpl</servlet-name>
    <servlet-class>com.node02.servlet.impl.ServletThreeImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletThreeImpl</servlet-name>
    <url-pattern>/servletThreeImpl</url-pattern>
</servlet-mapping>
```

- API用法

```java
public class ServletThreeImpl extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        // http://localhost:6003/servletThreeImpl?myName=cicada
        String method = request.getMethod();
        System.out.println("method="+method); // GET
        String requestURL = request.getRequestURL().toString();
        // http://localhost:6003/servletThreeImpl
        System.out.println("requestURL="+requestURL);
        String requestURI = request.getRequestURI();
        System.out.println("requestURI="+requestURI); // /servletThreeImpl
        String queryString = request.getQueryString() ;
        System.out.println("queryString="+queryString); // myName=cicada
        String myName = request.getParameter("myName");
        System.out.println("myName="+myName); // cicada
    }
}
```

# 九、ServletResponse接口

## 1、接口简介

HttpServletResponse继承自ServletResponse，封装了Http响应信息。客户端每个请求，服务器都会创建一个response对象，并传入给Servlet.service()方法。核心作用：1.设置响应头信息;2.发送状态码;3.设置响应正文;4.重定向；

## 2、代码案例

- 配置文件

```xml
<servlet>
    <servlet-name>servletFourImpl</servlet-name>
    <servlet-class>com.node02.servlet.impl.ServletFourImpl</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>servletFourImpl</servlet-name>
    <url-pattern>/servletFourImpl</url-pattern>
</servlet-mapping>
```

- API用法

```
public class ServletFourImpl extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=utf-8") ;
        response.setCharacterEncoding("UTF-8");
        response.setStatus(200) ;
        response.getWriter().print("Hello,知了");
    }
}
```

## 3、转发和重定向

- 转发

服务器端进行的页面跳转的控制 ;

```
request.getRequestDispatcher("/转发地址").forward(request, response);
```

- 重定向

服务端响应跳转信息，浏览器端进行的页面跳转 ;

```
response.sendRedirect("重定向地址");
```
- 转发和重定向对比

区别 | 转发 | 重定向
---|---|---
地址栏 | 不变 | 变化
跳转 | 服务端跳转 | 浏览器端跳转
请求次数 | 一次 | 两次
域中数据 | 不会丢失 | 丢失

**源码参考：** https://gitee.com/cicadasmile/java-base-parent