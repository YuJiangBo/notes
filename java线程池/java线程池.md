# java线程池

## ThreadPoolExecutor  线程池状态：

ThreadPoolExecutor使用int(32位)中的高三位表示线程池状态，低29位表示线程数量。这样做是为了减少cas原子操作次数，通过方法合并后就可以通过1次cas原子操作进行赋值。

| 状态名     | 高三位         | 接收新任务 | 处理阻塞队列任务 | 说明                                                     |
| ---------- | -------------- | ---------- | ---------------- | -------------------------------------------------------- |
| RUNNING    | 111 (-1 << 29) | Y          | Y                | 会接收新任务，也会处理阻塞队列任务                       |
| SHUTDOWN   | 000 (0 << 29)  | N          | Y                | 不会接收新任务，但会处理阻塞队列任务                     |
| STOP       | 001 (1 << 29)  | N          | N                | 不会接收新任务，会中断正在执行的任务，并抛弃阻塞队列任务 |
| TIDYING    | 010 (2 << 29)  | -          | -                | 任务全执行完毕，活动线程为0即将进入终结状态              |
| TERMINATED | 011 (3 << 29)  | -          | -                | 终结状态                                                 |

```java
// 合并成一个int的方法  rs=高三位  wc=低29位
private static int ctlOf(int rs, int wc) { return rs | wc; }
```



## ThreadPoolExecutor  构造方法参数：

```java
// 选择了参数最多的一个
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

- corePoolSize：核心线程数目（最多保留的线程数），保留在池中的线程数，即使它们是空闲的，除非设置allowCoreThreadTimeOut。
- maximumPoolSize：池中允许的最大线程数  （核心线程 + 救急线程）。
  - 总任务数大于核心线程正在执行的任务数加上组赛队列中最大任务capcity数时，多出的线程就会交给救急线程处理（线程总数要在最大线程数范围内）。执行完毕后过一段时间（默认为60s，时间由下面的keepAliveTime和unit控制，），救急线程就会自动关闭。
- keepAliveTime：当线程数大于核心时，这是多余的空闲线程在终止前等待新任务的最大时间（针对救急线程起作用）。
- unit：keepAliveTime参数的时间单位（针对救急线程起作用）。
- workQueue：阻塞队列。在任务执行之前用于保存任务的队列。这个队列将只保存execute方法提交的Runnable任务。
- threadFactory：当执行程序创建新线程时使用的工厂 - 创建时可以为线程起名字
- handler：拒绝策略，当执行因达到线程边界和队列容量而被阻塞时使用的处理程序（救急线程再不够时执行）

**RejectedExecutionHandler 拒绝策略：**

1. JDK默认实现：

   ![](C:\Users\jyu59\OneDrive - DXC Production\Desktop\learn\notes\java线程池\img\handler.png)

   - AbortPolicy：这是默认策略，让调用者抛出RejectedExecutionException异常。
   - CallerRunsPolicy：让调用者运行任务。
   - DiscardPolicy：放弃本次任务。
   - DiscardOldestPolicy：放弃队列中最早的任务，让本任务取而代之。

2. 其他工具实现：

   - Dubbo：抛出RejectedExecutionException异常之前会记录日志，并dump栈信息，方便记录问题。
   - Netty：创建一个新线程来执行任务
   - ActiveMQ：带超时等待（60s）尝试放入队列。
   - PinPoint：它使用一个拒绝策略链，会逐一尝试策略中的每种策略



## Executors创建的四种线程池

1. **newFixedThreadPool**：创建固定线程数的线程池。在任何时候，最多nThreads线程将是活动的处理任务。如果在所有线程都处于活动状态时提交额外的任务，它们将在队列中等待，直到有一个线程可用。如果任何线程在关闭之前因为执行失败而终止，那么如果需要执行后续任务，新的线程将会取代它。池中的线程将一直存在，直到显式关闭为止。

​		

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

// 可以指定ThreadFactory（可自定义），主要用来给线程起名字
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

​	特点：

- 核心线程数 = 最大线程数，即没有救急线程，所以也不需要超时时间。
- 阻塞队列是LinkedBlockingQueue无固定长度的链表，可以放任意长度的任务。

​	*适用于任务量已知，相对耗时的任务*



2. **newCachedThreadPool：**带缓冲线程池。

```java
// Integer.MAX_VALUE = 2³¹-1
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}


public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

特点：

- 核心线程数是0，最大线程数是2³¹ - 1，即创建的线程都是救急线程，空闲生存时间是60s，在这时间内可以去接下一个任务，救急线程数几乎可以无限创建。
- 队列采用了SynchronousQueue实现，它没有容量，没有消费的线程来取，放任务的线程是会阻塞的（理解为一手交钱，一手交货）。

*适合任务数比较密集，但每个任务执行时间较短的情况*



3. **newSingleThreadExecutor：**单线程线程池。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}


public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

​	特点：

- 只有一个核心线程，没有救急线程。
- 线程数固定为1，任务多余1时，会放入无界队列LinkedBlockingQueue，任务执行完毕，线程也不会被释放。
- 不是返回的ThreadPoolExecutor对象，而是返回的FinalizableDelegatedExecutorService。

​	*适用于希望多个任务排队执行*



区别：

- 与自己创建一个线程执行相比，自己创建一个线程串行执行任务，如果失败，就终止了，而newSingleThreadExecutor还会创建一个新的线程，以保证线程池的正常工作。
- newSingleThreadExecutor线程个数始终为1，不能修改
  - FinalizableDelegatedExecutorService 应用的是装饰器模式，只对外暴露了ExecutorService中的方法，不能调用ThreadPoolExecutor中特有的方法。
- 与newFixedThreadPool(1)相比，对外暴露的是ThreadPoolExecutor对象，可以强转后调用setCorePoolSize等方法，因此它的容量之后是可以修改的。



4. **newScheduledThreadPool：**周期性线程池

   ```java
   public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
       return new ScheduledThreadPoolExecutor(corePoolSize);
   }
   
   public static ScheduledExecutorService newScheduledThreadPool(
       int corePoolSize, ThreadFactory threadFactory) {
       return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
   }
   ```

​	特点：

- 没有救急线程，
- 缓存队列为DelayedWorkQueue。



jdk1.5之前的版本中更多的是借助Timer类来实现，

Timer和ScheduledThreadPoolExecutor的区别：

- Timer单线程运行，一旦任务执行缓慢，下一个任务就会推迟，而如果使用了ScheduledThreadPoolExecutor线程数可以自行控制

- 当Timer中的一个任务抛出异常时，会导致其他所有任务不在执行

- ScheduledThreadPoolExecutor可执行异步的任务，从而得到执行结果



ScheduledExecutorService接口继承了ExecutorService，在ExecutorService的基础上新增了以下几个方法：

- schedule方法：延时执行某个任务
- scheduleAtFixedRate方法：固定频率：每间隔固定时间就执行一次任务。注重频率。
- scheduleWithFixedDelay方法：固定延迟：任务之间的时间间隔，也就是说当上一个任务执行完成后，我会在固定延迟时间后出发第二次任务。注重距上次完成任务后的时间间隔。
