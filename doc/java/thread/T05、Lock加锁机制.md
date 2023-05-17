# 一、资源和加锁

## 1、场景描述

多线程并发访问同一个资源问题，假如线程A获取变量之后修改变量值，线程C在此时也获取变量值并且修改，两个线程同时并发处理一个变量，就会导致并发问题。

![](https://images.gitee.com/uploads/images/2022/0220/143309_33231061_5064118.png "05-1.png")

这种并行处理数据库的情况在实际的业务开发中很常见，两个线程先后修改数据库的值，导致数据有问题，该问题复现的概率不大，处理的时候需要对整个模块体系有概念，才能容易定位问题。

## 2、演示案例

```java
public class LockThread01 {
    public static void main(String[] args) {
        CountAdd countAdd = new CountAdd() ;
        AddThread01 addThread01 = new AddThread01(countAdd) ;
        addThread01.start();
        AddThread02 varThread02 = new AddThread02(countAdd) ;
        varThread02.start();
    }
}
class AddThread01 extends Thread {
    private CountAdd countAdd  ;
    public AddThread01 (CountAdd countAdd){
        this.countAdd = countAdd ;
    }
    @Override
    public void run() {
        countAdd.countAdd(30);
    }
}
class AddThread02 extends Thread {
    private CountAdd countAdd  ;
    public AddThread02 (CountAdd countAdd){
        this.countAdd = countAdd ;
    }
    @Override
    public void run() {
        countAdd.countAdd(10);
    }
}
class CountAdd {
    private Integer count = 0 ;
    public void countAdd (Integer num){
        try {
            if (num == 30){
                count = count + 50 ;
                Thread.sleep(3000);
            } else {
                count = count + num ;
            }
            System.out.println("num="+num+";count="+count);
        } catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

这里案例演示多线程并发修改count值，导致和预期不一致的结果，这是多线程并发下最常见的问题，尤其是在并发更新数据时。

出现并发的情况时，就需要通过一定的方式或策略来控制在并发情况下数据读写的准确性，这被称为并发控制，实现并发控制手段也很多，最常见的方式是资源加锁，还有一种简单的实现策略：修改数据前读取数据，修改的时候加入限制条件，保证修改的内容在此期间没有被修改。

# 二、锁的概念简介

## 1、锁机制简介

并发编程中一个最关键的问题，多线程并发处理同一个资源，防止资源使用的冲突一个关键解决方法，就是在资源上加锁：多线程序列化访问。锁是用来控制多个线程访问共享资源的方式，锁机制能够让共享资源在任意给定时刻只有一个线程任务访问，实现线程任务的同步互斥，这是最理想但性能最差的方式，共享读锁的机制允许多任务并发访问资源。

## 2、悲观锁

悲观锁，总是假设每次每次被读取的数据会被修改，所以要给读取的数据加锁，具有强烈的资源独占和排他特性，在整个数据处理过程中，将数据处于锁定状态，例如synchronized关键字的实现就是悲观机制。

![](https://images.gitee.com/uploads/images/2022/0220/143348_0630e01b_5064118.png "05-2.png")

悲观锁的实现，往往依靠数据库提供的锁机制，只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据，悲观锁主要分为共享读锁和排他写锁。

排他锁基本机制：又称写锁，允许获取排他锁的事务更新数据，阻止其他事务取得相同的资源的共享读锁和排他锁。若事务T对数据对象A加上写锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的写锁。

## 3、乐观锁

乐观锁相对悲观锁而言，采用更加宽松的加锁机制。悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性。但随之而来的就是数据库性能的大量开销，特别是对长事务的开销非常的占资源，乐观锁机制在一定程度上解决了这个问题。

![](https://images.gitee.com/uploads/images/2022/0220/143409_37cee9d1_5064118.png "05-3.png")

乐观锁大多是基于数据版本记录机制实现，为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个version字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号等于数据库表当前版本号，则予以更新，否则认为是过期数据。乐观锁机制在高并发场景下，可能会导致大量更新失败的操作。

乐观锁的实现是策略层面的实现：CAS(Compare-And-Swap)。当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能成功更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

## 4、机制对比

悲观锁本身的实现机制就以损失性能为代价，多线程争抢，加锁、释放锁会导致比较多的上下文切换和调度延时，加锁的机制会产生额外的开销，还有增加产生死锁的概率，引发性能问题。

乐观锁虽然会基于对比检测的手段判断更新的数据是否有变化，但是不确定数据是否变化完成，例如线程1读取的数据是A1，但是线程2操作A1的值变化为A2，然后再次变化为A1，这样线程1的任务是没有感知的。

悲观锁每一次数据修改都要上锁，效率低，写数据失败的概率比较低，比较适合用在写多读少场景。

乐观锁并未真正加锁，效率高，写数据失败的概率比较高，容易发生业务形异常，比较适合用在读多写少场景。

是选择牺牲性能，还是追求效率，要根据业务场景判断，这种选择需要依赖经验判断，不过随着技术迭代，数据库的效率提升，集群模式的出现，性能和效率还是可以两全的。

# 三、Lock基础案例

## 1、Lock方法说明

**lock**：执行一次获取锁，获取后立即返回；

**lockInterruptibly**：在获取锁的过程中可以中断；

**tryLock**：尝试非阻塞获取锁，可以设置超时时间，如果获取成功返回true，有利于线程的状态监控；

**unlock**：释放锁，清理线程状态；

**newCondition**：获取等待通知组件，和当前锁绑定；

## 2、应用案例

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
public class LockThread02 {
    public static void main(String[] args) {
        LockNum lockNum = new LockNum() ;
        LockThread lockThread1 = new LockThread(lockNum,"TH1");
        LockThread lockThread2 = new LockThread(lockNum,"TH2");
        LockThread lockThread3 = new LockThread(lockNum,"TH3");
        lockThread1.start();
        lockThread2.start();
        lockThread3.start();
    }
}
class LockNum {
    private Lock lock = new ReentrantLock() ;
    public void getNum (){
        lock.lock();
        try {
            for (int i = 0 ; i < 3 ; i++){
                System.out.println("ThreadName:"+Thread.currentThread().getName()+";i="+i);
            }
        } finally {
            lock.unlock();
        }
    }
}
class LockThread extends Thread {
    private LockNum lockNum ;
    public LockThread (LockNum lockNum,String name){
        this.lockNum = lockNum ;
        super.setName(name);
    }
    @Override
    public void run() {
        lockNum.getNum();
    }
}
```

这里多线程基于Lock锁机制，分别依次执行任务，这是Lock的基础用法，各种API的详解，下次再说。

## 3、与synchronized对比

基于synchronized实现的锁机制，安全性很高，但是一旦线程失败，直接抛出异常，没有清理线程状态的机会。显式的使用Lock语法，可以在finally语句中最终释放锁，维护相对正常的线程状态，在获取锁的过程中，可以尝试获取，或者尝试获取锁一段时间。

# 四、Lock体系结构

## 1、基础接口简介

Lock加锁相关结构中涉及两个使用广泛的基础API：ReentrantLock类和Condition接口，基本关系如下：

![](https://images.gitee.com/uploads/images/2022/0220/143528_63be1014_5064118.png "06-1.png")

**Lock接口**

Java并发编程中资源加锁的根接口之一，规定了资源锁使用的几个基础方法。

**ReentrantLock类**

实现Lock接口的可重入锁，即线程如果获得当前实例的锁，并进入任务方法，在线程没有释放锁的状态下，可以再次进入任务方法，特点：互斥排它性，即同一个时刻只有一个线程进入任务。

**Condition接口**

Condition接口描述可能会与锁有关联的条件变量，提供了更强大的功能，例如在线程的等待/通知机制上，Conditon可以实现多路通知和选择性通知。

## 2、使用案例

**生产消费模式**

写线程向容器中添加数据，读线程从容器获取数据，如果容器为空时，读线程等待。

```java
public class LockAPI01 {

    private static Lock lock = new ReentrantLock() ;
    private static Condition condition1 = lock.newCondition() ;
    private static Condition condition2 = lock.newCondition() ;

    public static void main(String[] args) throws Exception {
        List<String> dataList = new ArrayList<>() ;
        ReadList readList = new ReadList(dataList);
        WriteList writeList = new WriteList(dataList);
        new Thread(readList).start();
        TimeUnit.SECONDS.sleep(2);
        new Thread(writeList).start();
    }
    // 读数据线程
    static class ReadList implements Runnable {
        private List<String> dataList ;
        public ReadList (List<String> dataList){
            this.dataList = dataList ;
        }
        @Override
        public void run() {
            lock.lock();
            try {
                if (dataList.size() != 2){
                    System.out.println("Read wait...");
                    condition1.await();
                }
                System.out.println("ReadList WakeUp...");
                for (String element:dataList){
                    System.out.println("ReadList："+element);
                }
                condition2.signalAll();
            } catch (InterruptedException e){
                e.fillInStackTrace() ;
            } finally {
                lock.unlock();
            }
        }
    }
    // 写数据线程
    static class WriteList implements Runnable {
        private List<String> dataList ;
        public WriteList (List<String> dataList){
            this.dataList = dataList ;
        }
        @Override
        public void run() {
            lock.lock();
            try {
                dataList.add("Java") ;
                dataList.add("C++") ;
                condition1.signalAll();
                System.out.println("Write over...");
                condition2.await();
                System.out.println("Write WakeUp...");
            } catch (InterruptedException e){
                e.fillInStackTrace() ;
            } finally {
                lock.unlock();
            }
        }
    }
}
```

这个生产消费模式和生活中的点餐场景极为类似，用户下单，通知后厨烹饪，烹饪完成之后通知送餐。

**顺序执行模式**

既然线程执行可以互相通知，那也可以基于该机制实现线程的顺序执行，基本思路：在一个线程执行完毕后，基于条件唤醒下个线程。

```java
public class LockAPI02 {
    public static void main(String[] args) {
        PrintInfo printInfo = new PrintInfo() ;
        ExecutorService service =  Executors.newFixedThreadPool(3);
        service.execute(new PrintA(printInfo));
        service.execute(new PrintB(printInfo));
        service.execute(new PrintC(printInfo));
    }
}
class PrintA implements Runnable {
    private PrintInfo printInfo ;
    public PrintA (PrintInfo printInfo){
        this.printInfo = printInfo ;
    }
    @Override
    public void run() {
        printInfo.printA ();
    }
}
class PrintB implements Runnable {
    private PrintInfo printInfo ;
    public PrintB (PrintInfo printInfo){
        this.printInfo = printInfo ;
    }
    @Override
    public void run() {
        printInfo.printB ();
    }
}
class PrintC implements Runnable {
    private PrintInfo printInfo ;
    public PrintC (PrintInfo printInfo){
        this.printInfo = printInfo ;
    }
    @Override
    public void run() {
        printInfo.printC ();
    }
}
class PrintInfo {
    // 控制下个执行的线程
    private String info = "A";
    private ReentrantLock lock = new ReentrantLock();
    // 三个线程，三个控制条件
    Condition conditionA = lock.newCondition();
    Condition conditionB = lock.newCondition();
    Condition conditionC = lock.newCondition();
    public void printA (){
        try {
            lock.lock();
            while (!info.equals("A")) {
                conditionA.await();
            }
            System.out.print("A");
            info = "B";
            conditionB.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    public void printB (){
        try {
            lock.lock();
            while (!info.equals("B")) {
                conditionB.await();
            }
            System.out.print("B");
            info = "C";
            conditionC.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    public void printC (){
        try {
            lock.lock();
            while (!info.equals("C")) {
                conditionC.await();
            }
            System.out.print("C");
            info = "A";
            conditionA.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

该案例经常出现在多线程的面试题中，如何实现ABC的顺序打印问题，基本思路就是基于线程的等待通知机制，但是实现方式很多，上述只是其中一种方式。

# 五、读写锁机制

## 1、基础API简介

重入锁的排它特性决定了性能会产生瓶颈，为了提升性能问题，JDK中还有另一套读写锁机制。读写锁中维护一个共享读锁和一个排它写锁，在实际开发中，读的场景还是偏多的，所以读写锁可以很好的提高并发性。

读写锁相关结构中两个基础API：ReadWriteLock接口和ReentrantReadWriteLock实现类，基本关系如下：

![](https://images.gitee.com/uploads/images/2022/0220/143544_ebb58159_5064118.png "06-2.png")

**ReadWriteLock**

提供两个基础方法，readLock获取读机制锁，writeLock获取写机制锁。

**ReentrantReadWriteLock**

接口ReadWriteLock的具体实现，特点：基于读锁时，其他线程可以进行读操作，基于写锁时，其他线程读、写操作都禁止。

## 2、使用案例

**读写分离模式**

通过读写锁机制，分别向数据容器Map中写入数据和读取数据，以此验证读写锁机制。

```java
public class LockAPI03 {
    public static void main(String[] args) throws Exception {
        DataMap dataMap = new DataMap() ;
        Thread read = new Thread(new GetRun(dataMap)) ;
        Thread write = new Thread(new PutRun(dataMap)) ;
        write.start();
        Thread.sleep(2000);
        read.start();
    }
}
class GetRun implements Runnable {
    private DataMap dataMap ;
    public GetRun (DataMap dataMap){
        this.dataMap = dataMap ;
    }
    @Override
    public void run() {
        System.out.println("GetRun："+dataMap.get("myKey"));
    }
}
class PutRun implements Runnable {
    private DataMap dataMap ;
    public PutRun (DataMap dataMap){
        this.dataMap = dataMap ;
    }
    @Override
    public void run() {
        dataMap.put("myKey","myValue");
    }
}
class DataMap {
    Map<String,String> dataMap = new HashMap<>() ;
    ReadWriteLock rwLock = new ReentrantReadWriteLock() ;
    Lock readLock = rwLock.readLock() ;
    Lock writeLock = rwLock.writeLock() ;

    // 读取数据
    public String get (String key){
        readLock.lock();
        try{
            return dataMap.get(key) ;
        } finally {
            readLock.unlock();
        }
    }
    // 写入数据
    public void put (String key,String value){
        writeLock.lock();
        try{
            dataMap.put(key,value) ;
            System.out.println("执行写入结束...");
            Thread.sleep(10000);
        } catch (Exception e) {
            System.out.println("Exception...");
        } finally {
            writeLock.unlock();
        }
    }
}
```

说明：当put方法一直在睡眠状态时，因为写锁的排它性质，所以读方法是无法执行的。

# 六、基础工具类

**LockSupport简介**

LockSupprot定义一组公共静态方法，这些方法提供最基本的线程阻塞和唤醒功
能。

**基础方法**

park()：当前线程阻塞，当前线程被中断或调用unpark方法，park()方法中返回；

park(Object blocker)：功能同park()，传入Object对象，记录导致线程阻塞的阻塞对象，方便问题排查；

parkNanos(long nanos)：指定时间nanos内阻塞当前线程，超时返回；

unpark(Thread thread)：唤醒指定处于阻塞状态的线程；

**代码案例**

该流程在购物APP上非常常见，当你准备支付时放弃，会有一个支付失效，在支付失效期内可以随时回来支付，过期后需要重新选取支付商品。

```java
public class LockAPI04 {
    public static void main(String[] args) throws Exception {
        OrderPay orderPay = new OrderPay("UnPaid") ;
        Thread orderThread = new Thread(orderPay) ;
        orderThread.start();
        Thread.sleep(3000);
        orderPay.changeState("Pay");
        LockSupport.unpark(orderThread);
    }
}
class OrderPay implements Runnable {
    // 支付状态
    private String orderState ;
    public OrderPay (String orderState){
        this.orderState = orderState ;
    }
    public synchronized void changeState (String orderState){
        this.orderState = orderState ;
    }
    @Override
    public void run() {
        if (orderState.equals("UnPaid")){
            System.out.println("订单待支付..."+orderState);
            LockSupport.park(orderState);
        }
        System.out.println("orderState="+orderState);
        System.out.println("订单准备发货...");
    }
}
```

这里基于LockSupport中park和unpark控制线程状态，实现的等待通知机制。

**源码参考：** https://gitee.com/cicadasmile/java-base-parent