# 一、Executor框架简介

## 1、基础简介

Executor系统中，将线程任务提交和任务执行进行了解耦的设计，Executor有各种功能强大的实现类，提供便捷方式来提交任务并且获取任务执行结果，封装了任务执行的过程，不再需要Thread().start()方式，显式创建线程并关联执行任务。 

## 2、调度模型

线程被一对一映射为服务所在操作系统线程，启动时会创建一个操作系统线程；当该线程终止时，这个操作系统线程也会被回收。

![](https://images.gitee.com/uploads/images/2022/0220/144335_33d9a2c7_5064118.jpeg "08-1.jpg")

## 3、核心API结构

Executor框架包含的核心接口和主要的实现类如下图所示：

![](https://images.gitee.com/uploads/images/2022/0220/144345_5ce4158f_5064118.jpeg "08-2.jpg")

**线程池任务**：核心接口：Runnable、Callable接口和接口实现类；

**任务的结果**：接口Future和实现类FutureTask；

**任务的执行**：核心接口Executor和ExecutorService接口。在Executor框架中有两个核心类实现了ExecutorService接口，ThreadPoolExecutor和ScheduledThreadPoolExecutor。

# 二、用法案例

## 1、API基础

**ThreadPoolExecutor基础构造**

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {}
```

|参数名 | 说明 |
|:---|:---|
|corePoolSize | 线程池的核心大小，队列没满时，线程最大并发数|
|maximumPoolSize | 最大线程池大小，队列满后线程能够容忍的最大并发数|
|keepAliveTime | 空闲线程等待回收的时间限制|
|unit | keepAliveTime时间单位|
|workQueue | 阻塞的队列类型|
|threadFactory | 创建线程的工厂，一般用默认即可|
|handler | 超出工作队列和线程池时，任务会默认抛出异常|

## 2、初始化方法

```
ExecutorService ：Executors.newFixedThreadPool();
ExecutorService ：Executors.newSingleThreadExecutor();
ExecutorService ：Executors.newCachedThreadPool();

ThreadPoolExecutor ：new ThreadPoolExecutor() ;
```

通常情况下，线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式更加明确线程池的运行规则，规避资源耗尽的风险。

## 3、基础案例

```java
package com.multy.thread.block08executor;
import java.util.concurrent.*;

public class Executor01 {
    // 定义线程池
    private static ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(
                    3,10,5000,TimeUnit.SECONDS,
                    new SynchronousQueue<>(),Executors.defaultThreadFactory(),new ExeHandler());
    public static void main(String[] args) {
        for (int i = 0 ; i < 100 ; i++){
            poolExecutor.execute(new PoolTask(i));
            //带返回值：poolExecutor.submit(new PoolTask(i));
        }
    }
}
// 定义线程池任务
class PoolTask implements Runnable {

    private int numParam;

    public PoolTask (int numParam) {
        this.numParam = numParam;
    }
    @Override
    public void run() {
        try {
            System.out.println("PoolTask "+ numParam+" begin...");
            Thread.sleep(5000);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public int getNumParam() {
        return numParam;
    }
    public void setNumParam(int numParam) {
        this.numParam = numParam;
    }
}
// 定义异常处理
class ExeHandler implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable runnable, ThreadPoolExecutor executor) {
        System.out.println("ExeHandler "+executor.getCorePoolSize());
        executor.shutdown();
    }
}
```

**流程分析**

- 线程池中线程数小于corePoolSize时，新任务将创建一个新线程执行任务，不论此时线程池中存在空闲线程；
- 线程池中线程数达到corePoolSize时，新任务将被放入workQueue中，等待线程池中任务调度执行；
- 当workQueue已满，且maximumPoolSize>corePoolSize时，新任务会创建新线程执行任务；
- 当workQueue已满，且提交任务数超过maximumPoolSize，任务由RejectedExecutionHandler处理；
- 当线程池中线程数超过corePoolSize，且超过这部分的空闲时间达到keepAliveTime时，回收该线程；
- 如果设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize范围内的线程空闲时间达到keepAliveTime也将回收；

# 三、线程池应用

**应用场景**：批量账户和密码的校验任务，在实际的业务中算比较常见的，通过初始化线程池，把任务提交执行，最后拿到处理结果，这就是线程池使用的核心思想：节省资源提升效率。

```java
public class Executor02 {

    public static void main(String[] args) {
        // 初始化校验任务
        List<CheckTask> checkTaskList = new ArrayList<>() ;
        initList(checkTaskList);
        // 定义线程池
        ExecutorService executorService ;
        if (checkTaskList.size() < 10){
            executorService = Executors.newFixedThreadPool(checkTaskList.size());
        }else{
            executorService = Executors.newFixedThreadPool(10);
        }
        // 批量处理
        List<Future<Boolean>> results = new ArrayList<>() ;
        try {
            results = executorService.invokeAll(checkTaskList);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        // 查看结果
        for (Future<Boolean> result : results){
            try {
                System.out.println(result.get());
                // System.out.println(result.get(10000,TimeUnit.SECONDS));
            } catch (Exception e) {
                e.printStackTrace() ;
            }
        }
        // 关闭线程池
        executorService.shutdownNow();
    }

    private static void initList (List<CheckTask> checkTaskList){
        checkTaskList.add(new CheckTask("root","123")) ;
        checkTaskList.add(new CheckTask("root1","1234")) ;
        checkTaskList.add(new CheckTask("root2","1235")) ;
    }
}
// 校验任务
class CheckTask implements Callable<Boolean> {
    private String userName ;
    private String passWord ;
    public CheckTask(String userName, String passWord) {
        this.userName = userName;
        this.passWord = passWord;
    }
    @Override
    public Boolean call() throws Exception {
        // 校验账户+密码
        if (userName.equals("root") && passWord.equals("123")){
            return Boolean.TRUE ;
        }
        return Boolean.FALSE ;
    }
}
```

线程池主要用来解决线程生命周期开销问题和资源不足问题，通过线程池对多个任务线程重复使用，线程创建也被分摊到多个任务上，多数任务提交就有空闲的线程可以使用，所以消除线程频繁创建带来的开销。

**源码参考：** https://gitee.com/cicadasmile/java-base-parent