# 一、Fork/Join框架

Java提供Fork/Join框架用于并行执行任务，核心的思想就是将一个大任务切分成多个小任务，然后汇总每个小任务的执行结果得到这个大任务的最终结果。

这种机制策略在分布式数据库中非常常见，数据分布在不同的数据库的副本中，在执行查询时，每个服务都要跑查询任务，最后在一个服务上做数据合并，或者提供一个中间引擎层，用来汇总数据：

![](https://images.gitee.com/uploads/images/2022/0220/144041_80daab1c_5064118.png "07-1.png")

核心流程：切分任务，模块任务异步执行，单任务结果合并；在编程里面，通用的代码不多，但是通用的思想却随处可见。

# 二、核心API和方法

## 1、编码案例

基于1+2..+100的计算案例演示Fork/Join框架基础用法。

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;

public class ForkJoin01 {

    public static void main (String[] args) {
        int[] numArr = new int[100];
        for (int i = 0; i < 100; i++) {
            numArr[i] = i + 1;
        }
        ForkJoinPool pool = new ForkJoinPool();
        ForkJoinTask<Integer> forkJoinTask =
                pool.submit(new SumTask(numArr, 0, numArr.length));
        System.out.println("合并计算结果: " + forkJoinTask.invoke());
        pool.shutdown();
    }
}
/**
 * 线程任务
 */
class SumTask extends RecursiveTask<Integer> {
    /*
     * 切分任务块的阈值
     * 如果THRESHOLD=100
     * 输出：main【求和：(0...100)=5050】 合并计算结果: 5050
     */
    private static final int THRESHOLD = 100;
    private int arr[];
    private int start;
    private int over;

    public SumTask(int[] arr, int start, int over) {
        this.arr = arr;
        this.start = start;
        this.over = over;
    }

    // 求和计算
    private Integer sumCalculate () {
        Integer sum = 0;
        for (int i = start; i < over; i++) {
            sum += arr[i];
        }
        String task = "【求和：(" + start + "..." + over + ")=" + sum +"】";
        System.out.println(Thread.currentThread().getName() + task);
        return sum ;
    }

    @Override
    protected Integer compute() {
        if ((over - start) <= THRESHOLD) {
            return sumCalculate();
        }else {
            int middle = (start + over) / 2;
            SumTask left = new SumTask(arr, start, middle);
            SumTask right = new SumTask(arr, middle, over);
            left.fork();
            right.fork();
            return left.join() + right.join();
        }
    }
}
```

## 2、核心API说明

**ForkJoinPool**：线程池最大的特点就是分叉(fork)合并(join)模式，将一个大任务拆分成多个小任务，并行执行，再结合工作窃取算法提高整体的执行效率，充分利用CPU资源。

**ForkJoinTask**：运行在ForkJoinPool的一个任务抽象,可以理解为类线程但是比线程轻量的实体,在ForkJoinPool中运行的少量ForkJoinWorkerThread可以持有大量的ForkJoinTask和它的子任务，同时也是一个轻量的Future,使用时应避免较长阻塞或IO。

继承子类：

- RecursiveAction：递归无返回值的ForkJoinTask子类；
- RecursiveTask：递归有返回值的ForkJoinTask子类；

核心方法：

- fork()：在当前线程运行的线程池中创建一个子任务；
- join()：模块子任务完成的时候返回任务结果；
- invoke()：执行任务，也可以实时等待最终执行结果；

## 3、核心策略说明

**任务拆分**

![](https://images.gitee.com/uploads/images/2022/0220/144056_23867ffe_5064118.png "07-2.png")

ForkJoinPool基于分治算法，将大任务不断拆分下去，每个子任务再拆分一半，直到达到最阈值设定的任务粒度为止，并且把任务放到不同的队列里面，然后从最底层的任务开始执行计算，并且往上一层合并结果，这样用相对少的线程处理大量的任务。

**工作窃取算法**

![](https://images.gitee.com/uploads/images/2022/0220/144107_ac40d015_5064118.png "07-3.png")

大任务被分割为独立的子任务，并且子任务分别放到不同的队列里，并为每个队列创建一个线程来执行队列里的任务，假设线程A优先把分配到自己队列里的任务执行完毕，此时如果线程E对应的队列里还有任务等待执行，空闲的线程A会窃取线程E队列里任务执行，并且为了减少窃取任务时线程A和被窃取任务线程E之间的发生竞争，窃取任务的线程A会从队列的尾部获取任务执行，被窃取任务线程E会从队列的头部获取任务执行。

工作窃取算法的优点：线程间的竞争很少，充分利用线程进行并行计算，但是在任务队列里只有一个任务时，也可能会存在竞争情况。

# 三、应用案例分析

在后端系统的业务开发中，可用做权限校验，批量定时任务状态刷新等各种功能场景：

![](https://images.gitee.com/uploads/images/2022/0220/144121_a5b651bb_5064118.png "07-4.png")

如上图，假设数据的主键id分段如下，数据场景可能是数据源的连接信息，或者产品有效期类似业务，都可以基于线程池任务处理：

**权限校验**

基于数据源的连接信息，判断数据源是否可用，例如：判断连接是否可用，用户是否有库表的读写权限，在数据源多的情况下，基于线程池快速校验。

**状态刷新**

在定时任务中，经常见到状态类的刷新操作，例如判断产品是否在有效期范围内，在有效期范围之外，把数据置为失效状态，都可以利用线程池快速处理。

**源码参考：** https://gitee.com/cicadasmile/java-base-parent
