# 一、装配方式

Bean的概念：Spring框架管理的应用程序中，由Spring容器负责创建，装配，设置属性，进而管理整个生命周期的对象，称为Bean对象。

## 1、XML格式装配

Spring最传统的Bean的管理方式。

- 配置方式
```
<bean id="userInfo" class="com.spring.mvc.entity.UserInfo">
    <property name="name" value="cicada" />
</bean>
```
- 测试代码
```
ApplicationContext context01 = new ClassPathXmlApplicationContext("/bean-scan-02.xml");
UserInfo userInfo = (UserInfo)context01.getBean("userInfo") ;
System.out.println(userInfo.getName());
```

## 2、注解扫描

在实际开发中：通常使用注解 取代 xml配置文件。
- 常见注解
```
@Component <==> <bean class="Class">
@Component("id") <==> <bean id="id" class="Class">
@Repository ：Mvc架构中Dao层Bean的注解
@Service：Mvc架构中Service层Bean的注解
@Controller：Mvc架构中Controller层Bean的注解
```
- 使用案例

```
// 1、注解代码块
@Component("infoService")
public class InfoServiceImpl implements InfoService {
    @Override
    public void printName(String name) {
        System.out.println("Name："+name);
    }
}

// 2、配置代码块
@ComponentScan // 组件扫描注解
public class BeanConfig {

}
```
- 测试代码
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = BeanConfig.class)
public class Test01 {
    @Autowired
    private InfoService infoService ;
    @Test
    public void test1 (){
        infoService.printName("cicada");
        System.out.println(infoService==infoService);
    }
}
```

## 3、XML配置扫描

上面使用 ComponentScan 注解，也可在配置文件进行统一的配置，效果相同，还简化代码。
```
<context:component-scan base-package="com.spring.mvc" />
```

## 4、Java代码装配

这种基于Configuration注解，管理Bean的创建，在SpringBoot和SpringCloud的框架中，十分常见。
- 配置类代码
```
@Configuration // 配置类注解
public class UserConfig {
    @Bean
    public UserInfo userInfo (){
        System.out.println("userInfo...");
        return new UserInfo() ;
    }
}
```
- 测试代码
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = UserConfig.class)
public class Test03 {
    @Autowired
    private UserInfo userInfo ;
    @Autowired
    private UserInfo userInfo1 ;
    @Test
    public void test1 (){
        /*
         * userInfo...
         * true
         */
        System.out.println(userInfo==userInfo1);
    }
}
```

# 二、属性值设置

上面是Bean的装配几种常见方式，下面来看看Bean属性值设置，这里就基于Xml配置的方式。

## 1、基础类型和集合
- 配置代码
```
<!-- 配置Employee公共属性 -->
<bean id="emp1" class="com.spring.mvc.entity.Employee">
    <property name="name" value="cicada" />
    <property name="id" value="1" />
</bean>
<bean id="emp2" class="com.spring.mvc.entity.Employee">
    <property name="name" value="smile" />
    <property name="id" value="2" />
</bean>
<!-- 配置Department属性 -->
<bean id="department" class="com.spring.mvc.entity.Department">
    <!-- 普通属性值的注入 -->
    <property name="name" value="IT部门" />
    <!-- 给数组注入值 -->
    <property name="empName">
        <list>
            <value>empName1</value>
            <value>empName2</value>
            <value>empName3</value>
        </list>
    </property>
    <!-- 给List注入值：可以存放相同的值 -->
    <property name="empList">
        <list>
            <ref bean="emp1"/>
            <ref bean="emp2"/>
            <ref bean="emp1"/>
        </list>
    </property>
    <!-- 配置Set属性，相同的对象会被覆盖 -->
    <property name="empSet">
        <set>
            <ref bean="emp1"/>
            <ref bean="emp2"/>
            <ref bean="emp1"/>
        </set>
    </property>
    <!-- 配置Map属性,key相同的话，后面的值会覆盖前面的 -->
    <property name="empMap">
        <map>
            <entry key="1" value-ref="emp1" />
            <entry key="2" value-ref="emp2" />
            <entry key="2" value-ref="emp1" />
        </map>
    </property>
    <!-- 配置属性集合 -->
    <property name="pp">
        <props>
            <prop key="pp1">Hello</prop>
            <prop key="pp2">World</prop>
        </props>
    </property>
</bean>
```
- 测试代码
```
public class Test05 {
    @Test
    public void test01 (){
        ApplicationContext context = new ClassPathXmlApplicationContext("/bean-value-03.xml");
        Department department = (Department) context.getBean("department");
        System.out.println(department.getName());
        System.out.println("--------------------->String数组");
        for (String str : department.getEmpName()){
            System.out.println(str);
        }
        System.out.println("--------------------->List集合");
        for (Employee smp : department.getEmpList()){
            System.out.println(smp.getId()+":"+smp.getName());
        }
        System.out.println("--------------------->Set集合");
        for (Employee emp : department.getEmpSet()){
            System.out.println(emp.getId()+":"+emp.getName());
        }
        System.out.println("--------------------->Map集合");
        for (Map.Entry<String, Employee> entry : department.getEmpMap().entrySet()){
            System.out.println(entry.getKey()+":"+entry.getValue().getName());
        }
        System.out.println("--------------------->Properties");
        Properties pp = department.getPp();
        System.out.println(pp.get("pp1"));
        System.out.println(pp.get("pp2"));
    }
}
```

## 2、配置构造函数

根据配置的参数个数和类型，去映射并加载Bean的构造方法。
- 配置代码
```
<!-- 这里配置2个参数，所有调用2个参数的构造函数 -->
<bean id="employee" class="com.spring.mvc.entity.Employee">
    <constructor-arg index="0" type="java.lang.String" value="cicada"/>
    <constructor-arg index="1" type="int" value="1"/>
</bean>
```
- 测试代码
```
public class Test06 {
    @Test
    public void test01 (){
        ApplicationContext context = new ClassPathXmlApplicationContext("/bean-value-04.xml");
        Employee employee = (Employee) context.getBean("employee");
        System.out.println(employee.getId()+":"+employee.getName());
    }
}
```

## 3、配置继承关系

- 配置代码
```
<!-- 配置父类信息 -->
<bean id="student" class="com.spring.mvc.entity.Student">
    <property name="name" value="Spring" />
    <property name="age" value="22" />
</bean>
<!-- 配置子类信息 -->
<bean id="grade" class="com.spring.mvc.entity.Grade">
    <!-- 覆盖 -->
    <property name="name" value="Summer" />
    <property name="degree" value="大学" />
</bean>
```
- 测试代码
```
public class Test07 {
    @Test
    public void test01 (){
        ApplicationContext context = new ClassPathXmlApplicationContext("/bean-value-05.xml");
        Grade grade = (Grade) context.getBean("grade");
        /* Summer;0;大学  */
        System.out.println(grade.getName()+";"+grade.getAge()+";"+grade.getDegree());
    }
}
```

# 三、作用域

作用域：用于确定spring创建bean实例个数，比如单例Bean,原型Bean,等等。

类型 | 说明
---|---
singleton | IOC容器仅创建一个Bean实例，IOC容器每次返回的是同一个单例Bean实例，默认配置。
prototype | IOC容器可以创建多个Bean实例，每次返回的Bean都是新的实例。
request | 每次HTTP请求都会创建一个新的Bean，适用于WebApplicationContext环境。
session | 同一个HTTP Session共享一个Bean实例。不同HTTP Session使用不同的实例。
global-session | 同session作用域不同的是，所有的Session共享一个Bean实例。

# 四、生命周期

在Spring框架中Bean的生命周期非常复杂，过程大致如下：实例化，属性加载，初始化前后管理，销毁等。下面基于一个案例配置，会更加的清楚。

## 1、编写BeanLife类
```
public class BeanLife implements BeanNameAware {
    private String name ;
    public String getName() {
        return name;
    }
    public void setName(String name) {
        System.out.println("设置名称："+name);
        this.name = name;
    }
    @Override
    public void setBeanName(String value) {
        System.out.println("BeanNameAware..SetName："+value);
    }
    public void initBean() {
        System.out.println("初始化Bean..");
    }
    public void destroyBean() {
        System.out.println("销毁Bean..");
    }
    public void useBean() {
        System.out.println("使用Bean..");
    }
    @Override
    public String toString() {
        return "BeanLife [name = " + name + "]";
    }
}
```

## 2、定制加载过程

实现BeanPostProcessor接口。
```
public class BeanLifePostProcessor implements BeanPostProcessor {
    // 初始化之前对bean进行增强处理
    @Override
    public Object postProcessBeforeInitialization(Object obj, String beanName) throws BeansException {
        System.out.println("初始化之前..."+beanName);
        return obj ;
    }
    // 初始化之后对bean进行增强处理
    @Override
    public Object postProcessAfterInitialization(Object obj, String beanName) throws BeansException {
        System.out.println("初始化之后..."+beanName);
        // 改写Bean的名称
        if (obj instanceof BeanLife){
            BeanLife beanLife = (BeanLife)obj ;
            beanLife.setBeanName("myBeanLifeTwo");
            return beanLife ;
        }
        return obj ;
    }
}
```

## 3、配置文件
```
<!-- 加载Bean的处理器 -->
<bean class="com.spring.mvc.BeanLifePostProcessor" />
<!-- 指定初始化和销毁方法 -->
<bean id="beanLife" class="com.spring.mvc.entity.BeanLife"
    init-method="initBean" destroy-method="destroyBean">
    <property name="name" value="myBeanLifeOne" />
</bean>
```

## 4、测试过程

- 测试代码
```
public class Test08 {
    @Test
    public void test01 (){
        ApplicationContext context = new ClassPathXmlApplicationContext("/bean-value-06.xml");
        BeanLife beanLife = (BeanLife) context.getBean("beanLife");
        System.out.println("测试结果BeanLife："+beanLife.getName()) ;
        beanLife.useBean();
        // 关闭容器
        ((AbstractApplicationContext) context).close();
    }
}
```

- 输出结果
```
1、设置名称：myBeanLifeOne
2、BeanNameAware..SetName：beanLife
3、初始化之前...beanLife
4、初始化Bean..
5、初始化之后...beanLife
6、BeanNameAware..SetName：myBeanLifeTwo
7、测试结果BeanLife：myBeanLifeOne
8、使用Bean..
9、销毁Bean..
```
这里梳理Bean的生命周期，过程十分清晰。

**源码参考：** https://gitee.com/cicadasmile/spring-mvc-parent/tree/master/spring-base-node01