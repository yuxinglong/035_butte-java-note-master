# 一、AOP基础简介

## 1、切面编程简介

AOP全称：Aspect Oriented Programming，面向切面编程。通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。核心作用：可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的复用性和开发效率。AOP提供了取代继承和委托的一种新的方案，而且使用起来更加简洁清晰，是软件开发中的一个热点理念。

## 2、AOP术语

(1)、通知类型:Advice
```
前置通知[Before]：目标方法被调用之前;
返回通知[After-returning]：目标方法执行成功之后;
异常通知[After-throwing]：在目标方法抛出异常之后; 
后置通知[After]：目标方法完成之后;
环绕通知[Around]：在目标方法执行前后环绕通知;
```
(2)、连接点：JoinPoint

程序执行的某一个特定位置，如类初始前后，方法的运行前后。

(3)、切点:Pointcut

连接点是指那些在指定策略下可能被拦截到的方法。

(4)、切面：Aspect

切面由切点和通知的结合。

(5)、引入：Introduction

特殊的增强,为类添加一些属性和方法。

(6)、织入：Weaving

将增强添加到目标类的具体连接点上的过程。编译期织入，这要求使用特殊编译器；类装载期织入，这要求使用特殊的类加载器；动态代理织入，在运行期为目标类添加增强生成子类的方式，Spring采用的是动态代理织入，而AspectJ采用编译期织入和类装载期织入。

(7)、代理：Proxy

类被AOP织入后生成一个结果类，它是融合了原类和增强逻辑的代理类。

# 二、AOP编程实现方式

案例基于如下类进行：
```
public class Book {
    private String bookName ;
    private String author ;
}
public interface BookService {
    void addBook (Book book) ;
}
public class BookServiceImpl implements BookService {
    @Override
    public void addBook(Book book) {
        System.out.println(book.getBookName());
        System.out.println(book.getAuthor());
    }
}
```

## 1、JDK动态代理

```
public class BookAopProxyFactory {
    public static BookService createService() {
        // 目标类
        final BookService bookService = new BookServiceImpl() ;
        // 切面类
        final BookAspect bookAspect = new BookAspect();
        /*
         * 代理类：将目标类（切入点）和 切面类（通知） 结合
         */
        BookService proxyBookService = (BookService) Proxy.newProxyInstance(
                BookAopProxyFactory.class.getClassLoader(),
                bookService.getClass().getInterfaces(),
                new InvocationHandler() {
                    public Object invoke(Object proxy, Method method,
                                         Object[] args) throws Throwable {
                        // 前执行
                        bookAspect.before();
                        // 执行目标类的方法
                        Object obj = method.invoke(bookService, args);
                        // 后执行
                        bookAspect.after();
                        return obj;
                    }
                });
        return proxyBookService ;
    }
}
```

## 2、CgLib字节码增强

采用字节码增强框架cglib，在运行时创建目标类的子类，从而对目标类进行增强。
```
public class BookAopCgLibFactory {
    public static BookService createService() {
        // 目标类
        final BookService bookService = new BookServiceImpl() ;
        // 切面类
        final BookAspect bookAspect = new BookAspect();
        // 核心代理类
        Enhancer enhancer = new Enhancer();
        // 确定父类
        enhancer.setSuperclass(bookService.getClass());
        // 设置回调函数
        enhancer.setCallback(new MethodInterceptor() {
            public Object intercept(Object proxy, Method method,
                                    Object[] args,
                                    MethodProxy methodProxy) throws Throwable {
                bookAspect.before();
                Object obj = method.invoke(bookService, args);
                bookAspect.after();
                return obj;
            }
        });
        BookServiceImpl proxyService = (BookServiceImpl) enhancer.create();
        return proxyService ;
    }
}
```

## 3、Spring半自动代理

spring 创建代理对象，从spring容器中手动的获取代理对象。

- 配置文件
```
<!-- 创建目标类 -->
<bean id="bookService" class="com.spring.mvc.service.impl.BookServiceImpl" />
<!-- 创建切面类 -->
<bean id="myAspect" class="com.spring.mvc.config.BookAopSpringHalf" />
<!-- 创建代理类 -->
<bean id="proxyFactory" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="interfaces" value="com.spring.mvc.service.BookService" />
    <property name="target" ref="bookService" />
    <property name="interceptorNames" value="myAspect" />
</bean>
```
- 切面类
```
public class BookAopSpringHalf implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        System.out.println("Method Before ...");
        Object obj = methodInvocation.proceed();
        System.out.println("Method After ...");
        return obj;
    }
}
```


## 4、Spring全自动代理

从spring容器获得目标类，如果配置Aop，spring将自动生成代理。

- 配置文件
```
<!-- 创建目标类 -->
<bean id="bookService" class="com.spring.mvc.service.impl.BookServiceImpl" />
<!-- 创建切面类 -->
<bean id="myAspect" class="com.spring.mvc.config.BookAopSpringHalf" />
<!-- AOP编程配置 -->
<aop:config proxy-target-class="true">
    <aop:pointcut expression="execution(* com.spring.mvc.service.*.*(..))"
                  id="myPointCut"/>
    <aop:advisor advice-ref="myAspect" pointcut-ref="myPointCut"/>
</aop:config>
```

## 5、综合测试
```
@Test
public void test1 (){
    BookService bookService = BookAopProxyFactory.createService() ;
    Book book = new Book() ;
    book.setBookName("Spring实战");
    book.setAuthor("Craig Walls");
    bookService.addBook(book);
}
@Test
public void test2 (){
    BookService bookService = BookAopCgLibFactory.createService() ;
    Book book = new Book() ;
    book.setBookName("MySQL");
    book.setAuthor("Baron");
    bookService.addBook(book);
}
@Test
public void test3 (){
    String xmlPath = "spring-aop-half.xml";
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlPath);
    BookService bookService = (BookService) context.getBean("proxyFactory");
    Book book = new Book() ;
    book.setBookName("红楼梦");
    book.setAuthor("曹雪芹");
    bookService.addBook(book);
}
@Test
public void test4 (){
    String xmlPath = "spring-aop-all.xml";
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlPath);
    BookService bookService = (BookService) context.getBean("bookService");
    Book book = new Book() ;
    book.setBookName("西游记");
    book.setAuthor("吴承恩");
    bookService.addBook(book);
}
```

# 三、AspectJ切面编程

## 1、基础简介

AspectJ是一个基于Java语言的AOP框架，Spring2.0以后新增了对AspectJ切点表达式支持，通过JDK5注解技术，允许直接在类中定义切面，新版本Spring框架，推荐使用AspectJ方式来开发AOP编程。

## 2、XML配置方式

- 切面类
```
public class BookAopAspectJ {
    public void myBefore(JoinPoint joinPoint){
        System.out.println("前置通知：" + joinPoint.getSignature().getName());
    }
    public void myAfterReturning(JoinPoint joinPoint,Object ret){
        System.out.println("后置通知：" + joinPoint.getSignature().getName() + " , -->" + ret);
    }
    public Object myAround(ProceedingJoinPoint joinPoint) throws Throwable{
        System.out.println("环绕通知前");
        Object obj = joinPoint.proceed();
        System.out.println("环绕通知前后");
        return obj;
    }
    public void myAfterThrowing(JoinPoint joinPoint,Throwable e){
        System.out.println("抛出异常通知 ： " + e.getMessage());
    }
    public void myAfter(JoinPoint joinPoint){
        System.out.println("最终通知");
    }
}
```
- 配置文件
```
<!-- 创建目标类 -->
<bean id="bookService" class="com.spring.mvc.service.impl.BookServiceImpl" />
<!-- 创建切面类 -->
<bean id="myAspect" class="com.spring.mvc.config.BookAopAspectJ" />
<!-- 配置AOP编程 -->
<aop:config>
    <aop:aspect ref="myAspect">
        <aop:pointcut expression="execution(* com.spring.mvc.service.impl.BookServiceImpl.*(..))" id="myPointCut"/>
        <!-- 前置通知-->
        <aop:before method="myBefore" pointcut-ref="myPointCut"/>
        <!-- 后置通知 -->
        <aop:after-returning method="myAfterReturning" pointcut-ref="myPointCut" returning="ret" />
        <!-- 环绕通知 -->
        <aop:around method="myAround" pointcut-ref="myPointCut"/>
        <!-- 抛出异常 -->
        <aop:after-throwing method="myAfterThrowing" pointcut-ref="myPointCut" throwing="e"/>
        <!-- 最终通知 -->
        <aop:after method="myAfter" pointcut-ref="myPointCut"/>
    </aop:aspect>
</aop:config>
```
- 测试方法
```
@Test
public void test1 (){
    String xmlPath = "spring-aop-aspectj-01.xml";
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlPath);
    BookService bookService = (BookService) context.getBean("bookService");
    Book book = new Book() ;
    book.setBookName("三国演义");
    book.setAuthor("罗贯中");
    bookService.addBook(book);
}
```

## 3、注解扫描方式

- 配置文件
```
<!-- 开启类注解的扫描 -->
<context:component-scan base-package="com.spring.mvc.service.impl" />
<!-- 确定AOP注解生效 -->
<aop:aspectj-autoproxy />
<!-- 声明切面 -->
<bean id="myAspect" class="com.spring.mvc.config.AuthorAopAspectJ" />
<aop:config>
    <aop:aspect ref="myAspect" />
</aop:config>
```

- 注解切面类
```
@Component
@Aspect
public class AuthorAopAspectJ {
    @Pointcut("execution(* com.spring.mvc.service.impl.AuthorServiceImpl.*(..))")
    private void myPointCut(){
    }
    @Before("execution(* com.spring.mvc.service.impl.AuthorServiceImpl.*(..))")
    public void myBefore(JoinPoint joinPoint){
        System.out.println("前置通知：" + joinPoint.getSignature().getName());
    }
    @AfterReturning(value="myPointCut()" ,returning="ret")
    public void myAfterReturning(JoinPoint joinPoint,Object ret){
        System.out.println("后置通知：" +
                joinPoint.getSignature().getName() + " , -->" + ret);
    }
    @Around(value = "myPointCut()")
    public Object myAround(ProceedingJoinPoint joinPoint) throws Throwable{
        System.out.println("环绕通知前");
        Object obj = joinPoint.proceed();
        System.out.println("环绕通知前后");
        return obj;
    }
    @AfterThrowing(
            value="execution(* com.spring.mvc.service.impl.AuthorServiceImpl.*(..))",
            throwing="e")
    public void myAfterThrowing(JoinPoint joinPoint,Throwable e){
        System.out.println("抛出异常通知 ： " + e.getMessage());
    }
    @After("myPointCut()")
    public void myAfter(JoinPoint joinPoint){
        System.out.println("最终通知");
    }
}
```

- 测试方法
```
@Test
public void test2 (){
    String xmlPath = "spring-aop-aspectj-02.xml";
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlPath);
    AuthorService authorService = (AuthorService) context.getBean("authorService");
    System.out.println("作者："+authorService.getAuthor());
}
```

**源码参考：** https://gitee.com/cicadasmile/spring-mvc-parent/tree/master/spring-base-node03