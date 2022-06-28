

# JUC（java.util.concurrent）包的使用

官方文档：https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/package-summary.html 

> 重点类：
> 	1.AtomicInteger
> 	2.Semaphore
> 	3.CountDownLatch
> 	4.ThreadLocal
> 	5.ConcurrentHashMap
> 	6.ReentrantLock

## AtomicInteger

一个可以原子更新的int值。有关原子变量属性的描述，请参阅java.util.concurrent.atomic包规范。AtomicInteger用于原子递增计数器等应用程序，不能用作Integer的替代品。但是，这个类确实扩展了Number，以允许处理基于数字的类的工具和实用程序进行统一访问。



通过*volatile*修饰

```java
private volatile int value;
```

**volatile关键字的作用**：保证了变量的可见性（visibility）。被volatile关键字修饰的变量，如果值发生了变更，其他线程立马可见，避免出现脏读的现象。*volatile只能保证变量的可见性，不能保证对volatile变量操作的原子性*

**实例化**：

```java
// 无参
AtomicInteger atomicInteger = new AtomicInteger(); 	//初始化值为0
// 指定值
AtomicInteger atomicInteger2 = new AtomicInteger(2);
```



**方法**：

1. get()：返回当前值（int）

2. set(int newValue)：设置值

3. lazySet(int newValue)：最终设置为给定值。

   set()与lazySet(): set()会刷新缓存，保证缓存一致性，读到的值为最新值；lazySet不会刷新缓存，读到的值可能是旧值。

   ```java
   public class Atomic {
   
       AtomicInteger atomic = new AtomicInteger();
       public static void main(String[] args) {
   
           Atomic app = new Atomic();
           new Thread(() -> {
               for (int i = 0; i < 10; i++) {
                   app.atomic.lazySet(i); // 得到的结果中 get的值可能小于set的值（需要多试几遍）
                   //app.atomic.set(i); // 得到的结果中 get的值总是大于等于set的值
                   System.out.println("Set: " + i);
                   try {
                       Thread.sleep(100);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
           }).start();
   
           new Thread(() -> {
               for (int i = 0; i < 10; i++) {
                   synchronized (app.atomic) {
                       int counter = app.atomic.get();
                       System.out.println("Get: " + counter);
                   }
                   try {
                       Thread.sleep(100);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
           }).start();
       }
   }
   ```

4. getAndSet(int newValue)：原子地设置为给定值并返回旧值。

5. compareAndSet(int expect, int update)：如果当前值==预期值（expect），则自动将该值设置为给定的更新值（update）。

6. weakCompareAndSet(int expect, int update)：如果当前值==预期值，则自动将该值设置为给定的更新值。可能会错误地失败，并且不提供排序保证，所以它很少是比较andset的合适选择。

7. getAndIncrement()：将当前值原子地加1，并返回该值。

8. getAndDecrement()：将当前值原子地减1，并返回该值。

9. getAndAdd(int delta)：以原子方式将给定值（delta）添加到当前值，返回之前的值。

10. incrementAndGet()：将当前值原子地加1，返回更新后的值。

11. decrementAndGet()：将当前值原子地减1，返回更新后的值。

12. addAndGet(int delta)：以原子方式将给定值（delta）添加到当前值，返回更新后的值。

13. getAndUpdate(IntUnaryOperator updateFunction)：用应用给定函数的结果原子地更新当前值，返回前一个值。该函数应该没有副作用，因为当线程争用导致尝试更新失败时，可能会重新应用该函数。updateFunction - 一个没有副作用的函数，返回以前的值。

14. updateAndGet(IntUnaryOperator updateFunction)：用应用给定函数的结果原子地更新当前值，返回更新后的值。

    ```java
    public class Atomic2 {
        public static void main(String[] args) {
            AtomicInteger atomic = new AtomicInteger();
            AtomicInteger atomic1 = new AtomicInteger();
            IntUnaryOperator updateFunction = num -> num + 1;
            int oldValue = atomic.getAndUpdate(updateFunction);
            int current = atomic1.updateAndGet(updateFunction);
            System.out.println(oldValue);
            System.out.println(current);
        }
    }
    ```

15. getAndAccumulate(int x, IntBinaryOperator accumulatorFunction)：用将给定函数应用于当前值和给定值的结果原子地更新当前值，返回以前的值。函数的第一个参数是当前值，第二个参数是给定的更新。

16. accumulateAndGet(int x,IntBinaryOperator accumulatorFunction)：使用将给定函数应用于当前值和给定值的结果原子地更新当前值，返回更新后的值。函数的第一个参数是当前值，第二个参数是给定的更新。

    ```java
    class Atomic3 {
        public static void main(String[] args) {
            AtomicInteger atomic = new AtomicInteger();
            AtomicInteger atomic1 = new AtomicInteger(2);
            IntBinaryOperator binaryOperator = (x,y) -> x - y;
            int current = atomic.accumulateAndGet(0, binaryOperator);
            System.out.println(current);
            int oldValue = atomic1.getAndAccumulate(1, binaryOperator);
            System.out.println(oldValue);
            System.out.println(atomic1.get()); //atomic1.get() == atomic1 - 1
        }
    }
    ```

17. toString()：返回值为string类型

18. intValue()：返回值为int类型

19. longValue()：返回值为long类型

20. floatValue()：返回值为float类型

21. doubleValue()：返回值为double类型



## Semaphore

计数信号量。从概念上讲，信号量维护一组permits（许可/令牌）。如果有必要，每个人都会获取区块，直到获得许可，然后再获取。每次释放都会添加一个许可，可能会释放一个阻塞的获取者。但是，没有使用实际的permit对象;Semaphore只保留可用的数量的计数，并相应地进行操作。

信号量通常用于限制访问某些(物理或逻辑)资源的线程数量。有多个资源，也允许多个线程访问，只是限制同时访问共享资源的线程上限。（只是限制线程数量，不是限制资源数量）

理解：Semaphore就像停车场，permits就像车位数量，当线程获得了permits就像是获得了停车位，然后permits就减1。



```java
@Slf4j
class Atomic4 {

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                try {
                    semaphore.acquire(); // 获取许可permits,没有许可的线程，在此等待
                    log.debug("running");
                    Thread.sleep(1);
                    log.debug("end");
                    semaphore.release(); // 释放许可
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```



**方法：**

```java
acquire()  
获取一个令牌，在获取到令牌、或者被其他线程调用中断之前线程一直处于阻塞状态。

acquire(int permits)  
获取一个令牌，在获取到令牌、或者被其他线程调用中断、或超时之前线程一直处于阻塞状态。
    
acquireUninterruptibly() 
获取一个令牌，在获取到令牌之前线程一直处于阻塞状态（忽略中断）。
    
tryAcquire()
尝试获得令牌，返回获取令牌成功或失败，不阻塞线程。

tryAcquire(long timeout, TimeUnit unit)
尝试获得令牌，在超时时间内循环尝试获取，直到尝试获取成功或超时返回，不阻塞线程。

release()
释放一个令牌，唤醒一个获取令牌不成功的阻塞线程。

hasQueuedThreads()
等待队列里是否还存在等待线程。

getQueueLength()
获取等待队列里阻塞的线程数。

drainPermits()
清空令牌把可用令牌数置为0，返回清空令牌的数量。

availablePermits()
返回可用的令牌数量。
```

**实现原理：**

一、初始化

1. 当调用new Semaphore(2) 方法初始化时，默认会创建一个非公平的锁的同步阻塞队列。
2. 把初始令牌数量（2）赋值给同步队列的state状态，state的值就代表当前所剩余的令牌数量。

二、获取permits

当调用semaphore.acquire()方法时：

1. 当前线程会尝试去同步队列获取一个令牌，获取令牌的过程也就是使用原子的操作去修改同步队列的state ,获取一个令牌则修改为state=state-1。
2. 当计算出来的state<0，则代表令牌数量不足，（进入doAcquireSharedInterruptibly(arg)方法）此时会创建一个Node节点加入阻塞队列，挂起当前线程。
   - 封装一个Node节点，加入队列尾部
   - 在无限循环中，如果当前节点是头节点，就尝试获取信号
   - 不是头节点，在经过节点状态判断后，挂起当前线程
3. 当计算出来的state>=0，则代表获取令牌成功。

三、释放permits

当调用semaphore.release() 方法时

1. 线程会尝试释放一个令牌，释放令牌的过程也就是把同步队列的state修改为state=state+1的过程
2. 释放令牌成功之后，（进入doReleaseShared()方法）同时会唤醒同步队列中的一个线程。
   - 更新state加一
   - 唤醒等待队列头节点线程
3. 被唤醒的节点会重新尝试去修改state=state-1 的操作，如果state>=0则获取令牌成功，否则重新进入阻塞队列，挂起线程。



## CountdownLatch

一种同步辅助工具，它允许一个或多个线程等待，直到在其他线程中执行的一组操作完成。（用来进行线程同步协作，会让一个线程等待其他线程倒计时结束后才恢复运行）

CountDownLatch使用给定的计数进行初始化。由于调用countDown方法，await方法会阻塞直到当前计数为零，在此之后，所有等待的线程都会被释放，任何后续的await调用都会立即返回。这是一个一次性现象——计数器不能重置（同一个CountDownLatch对象，不能够重置计数）。



初始化:

```java
CountDownLatch startSignal = new CountDownLatch(3);
```

此时要想执行await方法后面的内容，就需要调用countDown方法，每次调用CountDownLatch倒计时都会减1，直到减为0，就可以执行await后续代码了。

示例：

```java
class Driver {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(3);
        for (int i = 0; i < 3; ++i) {
            new Thread(new Worker(startSignal, doneSignal)).start();
        }
        System.out.println("start");
        startSignal.countDown();      // startSignal倒计时减为0，即通知Worker可以执行await后的内容了
        System.out.println("doSomethingElse start: wait for others down");
        doneSignal.await();           // 等待doneSignal倒计时减为0，即等待Worker执行结束
        System.out.println("doSomethingElse down");
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;
    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }
    public void run() {
        try {
            startSignal.await();
            doWork();
            doneSignal.countDown();
        } catch (InterruptedException ex) {} // return;
    }

    void doWork() {
        System.out.println(Thread.currentThread().getName()+"do something");
    }
}
```

## CyclicBarrier

一种同步辅助工具，它允许一组线程全部等待彼此到达公共障碍点。CyclicBarrier在涉及固定大小的线程组的程序中很有用，这些线程组必须偶尔相互等待。该屏障称为循环屏障，因为它可以在释放等待线程后重用。

循环栅栏/屏障，用来进行线程协作，等待线程满足某个计数。初始化时设置计数个数，每个线程执行到某个需要”同步”的时刻调用await方法进行等待，当等待的线程满足计数个数时，继续执行。

CyclicBarrier 支持一个可选的 Runnable 命令，该命令在每个障碍点运行一次，在队列中的最后一个线程到达之后，但在释放任何线程之前。此屏障操作可用于在任何参与方继续之前更新共享状态。



```java
@Slf4j
class Atomic6 {

    public static void main(String[] args) {
        //线程数跟计数要一样：2，否则可能会造成两个task1执行时计数减为0了，task2执行时计数已经为0了。
        ExecutorService service = Executors.newFixedThreadPool(2);
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2,() -> {
            log.debug("task1 task2 finish");
        });

        for (int i = 0; i < 3; i++) {

            service.execute(() -> {
                log.debug("task1 begin");
                try {
                    Thread.sleep(1000);
                    cyclicBarrier.await(); // 2 - 1 = 1
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                log.debug("task1 finish");
            });

            service.execute(() -> {
                log.debug("task2 begin");
                try {
                    Thread.sleep(1000);
                    cyclicBarrier.await(); // 1 - 1 = 0
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                log.debug("task2 finish");
            });
        }
    }
}
```