# 线程池的正确创建

*`Executors`*创建线程池的方法，创建出来的线程池都实现了`ExecutorService`接口。常用方法有以下几个：

```java
  1.      ExecutorService executorService1 = Executors.newSingleThreadExecutor();
  2.      ExecutorService executorService2 = Executors.newCachedThreadPool();
  3.      ExecutorService executorService3 = Executors.newFixedThreadPool(2);
  4.      ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(2);
```

看起来功能还是比较强大的，又用到了工厂模式、又有比较强的扩展性，重要的是用起来还比较方便

但是，对于装了阿里规范扫描插件的同学来说会提示以下问题,使用Executors创建线程池可能会导致OOM(OutOfMemory ,内存溢出),但是并没有说明为什么，那么接下来我们就来看一下到底为什么不建议使用Executors？

![avatar](https://picture.zhanghong110.top/docsify/16411908661476.png)

> 我们来追寻一下原因，我们先分个组，首先看下`newSingleThreadExecutor`和`newFixedThreadPool`

看一下它两的实现

```java

   newSingleThreadExecutor：
  
   public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    
    
    newFixedThreadPool：
    
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }


```

可以看到都是用`LinkedBlockingQueue`实现。

Java中的阻塞队列`BlockingQueue`主要有两种实现，分别是`ArrayBlockingQueue` 和`LinkedBlockingQueue`。

`ArrayBlockingQueue`是一个用数组实现的有界阻塞队列，必须设置容量。

`LinkedBlockingQueue`是一个用链表实现的有界阻塞队列，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为`Integer.MAX_VALUE`。

这边调用它两的无参构造我们看一下

```java
    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
```

可以看到不设置长度的话默认长度是`Integer.MAX_VALUE`,问题就出在这，`newFixedThreadPool`中创建`LinkedBlockingQueue`时，并未指定容量。此时，`LinkedBlockingQueue`就是一个无边界队列，对于一个无边界队列来说，是可以不断的向队列中加入任务的，这种情况下, 理论上可以无限添加任务到线程池,如果超过核心线程的数量，将会放入队列中。就有可能因为单个任务执行时间过长导致任务堆积过多而导致内存溢出问题。



> 下一个我们来看`newScheduledThreadPool`。这个通常用来执行定时的任务。

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
```

首先其`corePoolSize`只是当前设置的值，最大值依然是`Integer.MAX_VALUE`

可以看到使用了`DelayedWorkQueue`,我们瞅一眼它的grow方法

```java
// 数组的初始容量为16 设置为16的原因跟hashmap中数组容量为16的原因一样
private static final int INITIAL_CAPACITY = 16;
// 用于记录RunableScheduledFuture任务的数组
private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
private final ReentrantLock lock = new ReentrantLock();
// 当前队列中任务数,即队列深度
private int size = 0;
// leader线程用于等待队列头部任务,
private Thread leader = null;
// 当线程成为leader时,通知其他线程等待
private final Condition available = lock.newCondition();

 
 private void grow() {
            int oldCapacity = queue.length;
            int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
            if (newCapacity < 0) // overflow
                newCapacity = Integer.MAX_VALUE;
            queue = Arrays.copyOf(queue, newCapacity);
        }
```

可见这货延迟队列中的数组支持动态扩容 单个任务时间过长时即可导致该任务大量叠加最后OOM



> 最后是`newCachedThreadPool`。

创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。

老规矩看一下实现

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

可以看到本身`maximumPoolSize`就能支持到`Integer.MAX_VALUE`所以呢这个本身就能缓存巨大的线程，导致OOM



*总结一下的话就是`newSingleThreadExecutor`和`newFixedThreadPool`的阻塞队列为极限是`Integer.MAX_VALUE`的有界阻塞队列，如果超过核心线程的数量，将会放入队列中，单个任务时间过长时会导致任务量过大时可能会导致内存溢出。`newScheduledThreadPool`允许的创建最大线程数量为`Integer.MAX_VALUE`，且延迟队列中的数组支持动态扩容 单个任务时间过长时即可导致该任务大量叠加最后OOM。`newCachedThreadPool`允许的创建最大线程数量为`Integer.MAX_VALUE`，可以支持无限大的缓存导致OOM。*



> 那么我们应当如何正确的创建线程池呢



说白了,避免使用Executors创建线程池，主要是避免使用其中的默认实现(可能会导致OOM)

那么我们可以自己直接调用`ThreadPoolExecutor`的构造函数来自己创建线程池。在创建的同时，给`BlockQueue`指定容量就可以了。

我们看下这个构造

```java

    /**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters, the
     * {@linkplain Executors#defaultThreadFactory default thread factory}
     * and the {@linkplain ThreadPoolExecutor.AbortPolicy
     * default rejected execution handler}.
     *
     * <p>It may be more convenient to use one of the {@link Executors}
     * factory methods instead of this general purpose constructor.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

解释下各个参数

`corePoolSiz`e – 池中要保留的线程数，即使它们处于空闲状态，除非设置了 `allowCoreThreadTimeOut`

 `maximumPoolSize` – 池中允许的最大线程数 

`keepAliveTime` – 当线程数大于核心时，这是多余空闲线程在终止前等待新任务的最长时间。

 `uni`t – `keepAliveTime` 参数的时间单位

`workQueue` – 用于在执行任务之前保存任务的队列。这个队列将只保存 execute 方法提交的 Runnable 任务。

举个栗子

```java
ExecutorService executor = new ThreadPoolExecutor(10, 10, 60L, TimeUnit.SECONDS, new ArrayBlockingQueue(10));
```

即可，另外这个类还有两个不同参的构造，多了

```java
ThreadFactory threadFactory
RejectedExecutionHandler handler
```

一个是线程工厂，用来创建线程，主要是用来给线程起名字

还有一个是拒绝策略，当线程池被耗尽或者队列满了的时候会调用,默认的实现是`**AbortPolicy**`,如果线程池队列满了丢掉这个任务并且抛出`RejectedExecutionException`异常。