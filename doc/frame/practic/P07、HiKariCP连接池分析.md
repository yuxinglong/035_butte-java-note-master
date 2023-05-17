> 池塘里养：Connection；

# 一、设计与原理

## 1、基础案例

HiKariCP作为SpringBoot2框架的默认连接池，号称是跑的最快的连接池，数据库连接池与之前两篇提到的线程池和对象池，从设计的原理上都是基于池化思想，只是在实现方式上有各自的特点；首先还是看HiKariCP用法的基础案例：

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.Statement;

public class ConPool {
    private static HikariConfig buildConfig (){
        HikariConfig hikariConfig = new HikariConfig() ;
        // 基础配置
        hikariConfig.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/junit_test?characterEncoding=utf8");
        hikariConfig.setUsername("root");
        hikariConfig.setPassword("123456");
        // 连接池配置
        hikariConfig.setPoolName("dev-hikari-pool");
        hikariConfig.setMinimumIdle(4);
        hikariConfig.setMaximumPoolSize(8);
        hikariConfig.setIdleTimeout(600000L);
        return hikariConfig ;
    }
    public static void main(String[] args) throws Exception {
        // 构建数据源
        HikariDataSource dataSource = new HikariDataSource(buildConfig()) ;
        // 获取连接
        Connection connection = dataSource.getConnection() ;
        // 声明SQL执行
        Statement statement = connection.createStatement();
        ResultSet resultSet = statement.executeQuery("SELECT count(1) num FROM jt_activity") ;
        // 输出执行结果
        if (resultSet.next()) {
            System.out.println("query-count-result："+resultSet.getInt("num"));
        }
    }
}
```

## 2、核心相关类

- **HikariDataSource类**：汇集数据源描述的相关信息，例如配置、连接池、连接对象、状态管理等；
- **HikariConfig类**：维护数据源的配置管理，以及参数校验，例如userName、passWord、minIdle、maxPoolSize等；
- **HikariPool类**：提供对连接池与池中对象管理的核心能力，并实现池相关监控数据的查询方法；
- **ConcurrentBag类**：抛弃了常规池中采用的阻塞队列作为容器的方式，自定义该并发容器来存储连接对象；
- **PoolEntry类**：拓展连接对象的信息，例如状态、时间等，方便容器中追踪这些实例化对象；

通过对连接池中几个核心类的分析，也能直观地体会到该源码的设计原理，与上篇总结的对象池应用有异曲同工之妙，只是不同的组件不同的开发者在实现的时候，都具备各自的抽象逻辑。

## 3、加载逻辑

![](https://images.gitee.com/uploads/images/2022/0417/163849_84c485fe_5064118.png "07-1.png")

通过配置信息去构建数据源描述，在构造方法中基于配置再去实例化连接池，在HikariPool的构造中，实例化ConcurrentBag容器对象；下面再从源码层面分析实现细节。

# 二、容器分析

## 1、容器结构

容器ConcurrentBag类提供PoolEntry类型的连接对象存储，以及基本的元素管理能力，对象的状态描述；虽然被HikariPool对象池类所持有，但是实际的操作逻辑是在该类中；

**1.1 基础属性**

其中最为核心的是`sharedList`共享集合、`threadList`线程级缓存、`handoffQueue`即时队列；

```java
// 共享对象集合，存放数据库连接
private final CopyOnWriteArrayList<T> sharedList;
// 缓存线程级连接对象，会被优先使用，避免被争抢
private final ThreadLocal<List<Object>> threadList;
// 等待获取连接的线程数
private final AtomicInteger waiters;
// 标记是否关闭
private volatile boolean closed;
// 即时处理连接的队列，当有等待线程时，通过该队列将连接分配给等待线程
private final SynchronousQueue<T> handoffQueue;
```

**1.2 状态描述**

在ConcurrentBag类中的IConcurrentBagEntry内部接口，被PoolEntry类实现，该接口定义连接对象的状态：

- STATE_NOT_IN_USE：未使用，即闲置中；
- STATE_IN_USE：使用中；
- STATE_REMOVED：被废弃；
- STATE_RESERVED：保留态，中间状态，用于尝试驱逐连接对象时；

## 2、包装对象

容器的基本能力是用来存储连接对象的，而对象的管理则需要很多扩展的跟踪信息，以有效的完成各种场景下的识别，此时就需要借助包装类的引入；

```java
// 业务真正使用的连接对象
Connection connection;
// 最近访问时间
long lastAccessed;
// 最近借出时间
long lastBorrowed;
// 状态描述
private volatile int state = 0;
// 是否驱逐
private volatile boolean evict;
// 生命周期结束时的调度任务
private volatile ScheduledFuture<?> endOfLife;
// 连接生成的Statement对象
private final FastList<Statement> openStatements;
// 池对象
private final HikariPool hikariPool;
```

这里需要注意FastList类实现List接口，为HiKariCP组件自定义，相比ArrayList类，出于对性能的追求，在元素的管理时，去掉诸多的范围校验。

# 三、对象管理

基于连接池的常规用法，来看看连接对象具体是如何管理，比如被借出，被释放，被废弃等，以及这些操作下对象的状态转换过程；

## 1、初始化

上文**加载逻辑**的描述中，已经提到在构建数据源的时候，会根据配置实例化连接池，在初始化的时候，基于两个核心切入点来分析源码：1.实例化多少连接对象、2.连接对象转换包装对象；

在连接池的构造中执行了`checkFailFast`方法，在该方法内执行MinIdle最小空闲数的判断，如果大于0，则创建一个包装对象并放入容器中；

![](https://images.gitee.com/uploads/images/2022/0417/163901_f3c3b5fb_5064118.png "07-2.png")

```java
public HikariPool(final HikariConfig config) ;
private void checkFailFast() {
    final PoolEntry poolEntry = createPoolEntry();
    if (config.getMinimumIdle() > 0) {
        connectionBag.add(poolEntry);
    }
}
```

需要注意两个问题，创建的连接包装对象，初始状态是0即闲置中；另外虽然案例中设置`MinIdle=4`的值，但是这里的判断大于0，也只在容器中预先放入一个空闲对象；

## 2、借用对象

从池中获取连接对象时，实际调用的是容器类中的`borrow`方法：

```java
public Connection HikariPool.getConnection(final long hardTimeout) throws SQLException ;
public T ConcurrentBag.borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException ;
```

![](https://images.gitee.com/uploads/images/2022/0417/163915_a09c8b24_5064118.png "07-3.png")

在执行`borrow`方法时，涉及如下几个核心步骤与逻辑：

```java
public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
{
    // 遍历本地线程缓存
    final List<Object> list = threadList.get();
    for (int i = list.size() - 1; i >= 0; i--) {
       final Object entry = list.remove(i);
       final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
       if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) { }
    }
    // 增加等待线程数
    final int waiting = waiters.incrementAndGet();
    try {
        // 遍历Shared共享集合
        for (T bagEntry : sharedList) {
           if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) { }
        }
        // 一定时间内轮询handoff队列
        listener.addBagItem(waiting);
        timeout = timeUnit.toNanos(timeout);
        do {
           final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
        } 
    } finally {
        // 减少等待线程数
       waiters.decrementAndGet();
    }
}
```

- 首先反向遍历本地线程缓存，如果存在空闲连接，则返回该对象；如果没有则寻找共享集合；
- 遍历Shared共享集合前，会标记等待线程数加1，如果存在空闲连接则直接返回；
- 当Shared共享集合中也没有空闲连接时，这时当前线程进行一定时间的`handoffQueue`队列轮询，可能会有资源的释放，也可能是新添加的资源；

注意这里在遍历集合时，取出的对象都会对状态进行判断和更新，如果得到空闲对象，会更新为`IN_USE`状态，然后返回；

## 3、释放对象

从池中释放连接对象时，实际调用的是容器类中的`requite`方法：

```java
void HikariPool.recycle(final PoolEntry poolEntry) ;
public void ConcurrentBag.requite(final T bagEntry) ;
```

![](https://images.gitee.com/uploads/images/2022/0417/163926_09323db9_5064118.png "07-4.png")

在释放连接对象时，首先更新对象状态为空闲，然后判断当前是否有等待的线程，在`borrow`方法中等待线程会进入一定时间的轮询，如果没有的话则把对象放入本地线程缓存中：

```java
public void requite(final T bagEntry) {
    // 更新状态
    bagEntry.setState(STATE_NOT_IN_USE);
    // 等待线程判断
    for (int i = 0; waiters.get() > 0; i++) {
        if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) { }
    }
    // 本地线程缓存
    final List<Object> threadLocalList = threadList.get();
    if (threadLocalList.size() < 50) {
        threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry);
    }
}
```

注意这里涉及到连接对象的状态从使用中转为`NOT_IN_USE`空闲；`borrow`与`requite`作为连接池中两个核心方法，负责资源创建与回收；

**最后**本篇文章并没有站在HiKariCP组件的整体设计上构思，只是分析连接池这冰山一角，尽管只是部分源码，但是已经足够彰显出作者对于性能的极致追求，比如：本地线程缓存、自定义容器类型、FastList等；能被普遍采用必然存在诸多支撑的理由。

# 四、参考源码

```
应用仓库：
https://gitee.com/cicadasmile/butte-flyer-parent

组件封装：
https://gitee.com/cicadasmile/butte-frame-parent
```