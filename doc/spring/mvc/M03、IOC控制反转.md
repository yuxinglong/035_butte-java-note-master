# 一、IOC控制反转

## 1、IOC容器思想

Java系统中对象耦合关系十分复杂，系统的各模块之间依赖，微服务模块之间的相互调用请求，都是这个道理。降低系统模块之间、对象之间、微服务的服务之间耦合度，是软件工程核心问题之一。因为Spring框架中核心思想就是IOC控制反转，用来实现对象之间的解耦。

## 2、控制反转

![](https://images.gitee.com/uploads/images/2022/0123/230703_32950c4b_5064118.png "03-1.png")

- 传统方式

对象A如果想使用对象B的功能方法，在需要的时候创建对象B的实例，调用需要的方法，对对象B有主动的控制权。

- IOC容器

当使用IOC容器之后，对象A和B之间失去了直接联系，对象A如果想使用对象B的功能方法，IOC容器会自动创建一个对象B实例注入到对象A需要的功能模块中，这样对象A失去了主动控制权，也就是控制反转了。

## 3、依赖注入

IOC给对象直接建立关系的动作，称为DI依赖注入(Dependency Injection);依赖：对象A需要使用对象B的功能，则称对象A依赖对象B。注入：在对象A中实例化对象B，从而使用对象B的功能，该动作称为注入。

# 二、IOC容器案例

## 1、买票乘车场景

- 简单乘车类
```
public class ByBus {
    // 方式一：直接实例化
    // private BuyTicket buyTicket = new BuyTicket () ;

    private BuyTicket buyTicket ;
    public BuyTicket getBuyTicket() {
        return buyTicket;
    }
    public void setBuyTicket(BuyTicket buyTicket) {
        this.buyTicket = buyTicket;
    }
    public void takeBus (){
        String myTicket = this.getBuyTicket().getTicket() ;
        if (myTicket.equals("ticket")){
            System.out.println("乘车");
        }
    }

}
```
- 简单买票类
```
public class BuyTicket {
    public String getTicket (){
        return "ticket" ;
    }
}
```

## 2、Spring配置文件

这里用过Spring配置文件，给乘车类中，注入买票类，进而完成完整动作。
```
<bean id="byBus" class="com.spring.mvc.entity.ByBus">
    <property name="buyTicket" ref="buyTicket" />
</bean>
<bean id="buyTicket" class="com.spring.mvc.entity.BuyTicket"/>
```

## 3、测试代码

```
public class Test1 {
    @Test
    public void test01 (){
        ApplicationContext context = 
        new ClassPathXmlApplicationContext("/ioc-contain-01.xml");
        ByBus byBus = (ByBus) context.getBean("byBus");
        byBus.takeBus();
    }
}
```

# 三、核心API总结

针对上面用到的几个核心API进行说明，后续持续总结。

**1、BeanFactory**

这是一个工厂，用于生成任意bean。采取延迟加载，第一次getBean时才会初始化Bean。

**2、ApplicationContext**

是BeanFactory的子接口，功能更强大。（国际化处理、事件传递、Bean自动装配、各种不同应用层的Context实现）。当配置文件被加载，就进行对象实例化。

**3、ClassPathXmlApplicationContext** 

用于加载classpath（类路径、src）下的xml加载xml运行时位置：/WEB-INF/classes/...xml

**4、FileSystemXmlApplicationContext**

用于加载指定盘符下的xml加载xml运行时位置：/WEB-INF/...xml，通过ServletContext.getRealPath()获得具体盘符配置。

**源码参考：** https://gitee.com/cicadasmile/spring-mvc-parent/tree/master/spring-base-node02
