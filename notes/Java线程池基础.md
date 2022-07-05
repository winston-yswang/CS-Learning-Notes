## 1、Executor框架

使用线程池可以降低资源消耗、提高响应速度以及提高线程的可管理性。

Executor 框架是 Java5 之后引进的，不仅包括了线程池的管理，还提供了线程工厂、队列以及拒绝策略等。

![任务的执行相关接口](pics\任务的执行相关接口.png)

Executor框架结构由三大部分组成：

- 任务（`Runnable` / `Callable`）：执行任务需要实现的 **`Runnable` 接口** 或 **`Callable`接口**。**`Runnable` 接口**或 **`Callable` 接口** 实现类都可以被 **`ThreadPoolExecutor`** 或 **`ScheduledThreadPoolExecutor`** 执行;
- 任务的执行（`Executor`）：使用最多的是 `ThreadPoolExecutor`;
- 异步计算的结果（`Future`）：`Future` 接口以及 `Future` 接口的实现类 `FutureTask` 类都可以代表异步计算的结果。

Executor框架的使用示意图：

![Executor框架的使用示意图](pics\Executor框架的使用示意图.png)

1. 主线程首先创建一个实现了`Runnable` 或者 `Callable` 接口的任务对象；
2. 把任务对象直接交给 `ExecutorService` 执行
3. 如果执行 `ExecutorService.submit(..)`，`ExecutorService` 将返回一个实现了`Future` 接口的对象；
4. 最后，主线程可以执行 `FutureTask.get()` 方法来等待执行完成。



## 2、ThreadPoolExecutor

### 2.1 线程池的创建

线程池可以通过`ThreadPoolExecutor`来创建，我们来看一下它的构造函数，`ThreadPoolExecutor` 类中提供的四个构造方法，下面是最长的那个，其余三个都是在这个构造方法的基础上产生。

```java
/**
 * 用给定的初始参数创建一个新的ThreadPoolExecutor。
 */
public ThreadPoolExecutor(int corePoolSize,//线程池的核心线程数量
                          int maximumPoolSize,//线程池的最大线程数
                          long keepAliveTime,//当线程数大于核心线程数时，多余的空闲线程存活的最长时间
                          TimeUnit unit,//时间单位
                          BlockingQueue<Runnable> workQueue,//任务队列，用来储存等待执行任务的队列
                          ThreadFactory threadFactory,//线程工厂，用来创建线程，一般默认即可
                          RejectedExecutionHandler handler//拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
                         ) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

**`ThreadPoolExecutor` 3 个最重要的参数：**

- **`corePoolSize` :** 核心线程数线程数定义了最小可以同时运行的线程数量。
- **`maximumPoolSize` :** 当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
- **`workQueue`:** 当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

`ThreadPoolExecutor`其他常见参数:

1. **`keepAliveTime`**:当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁；
2. **`unit`** : `keepAliveTime` 参数的时间单位。
3. **`threadFactory`** :executor 创建新线程的时候会用到。
4. **`handler`** :饱和策略。关于饱和策略下面单独介绍一下。

![线程池各个参数之间的关系](pics\线程池各个参数之间的关系.png)

**`ThreadPoolExecutor` 饱和策略定义:**

如果当前同时运行的线程数量达到最大线程数量并且队列也已经被放满了任务时，`ThreadPoolTaskExecutor` 定义一些策略:

- **`ThreadPoolExecutor.AbortPolicy`** ：抛出 `RejectedExecutionException`来拒绝新任务的处理。
- **`ThreadPoolExecutor.CallerRunsPolicy`** ：调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
- **`ThreadPoolExecutor.DiscardPolicy`** ：不处理新任务，直接丢弃掉。
- **`ThreadPoolExecutor.DiscardOldestPolicy`** ： 此策略将丢弃最早的未处理的任务请求。



### 2.2 任务执行

线程池执行流程，即对应execute()方法：

![图解线程池实现原理](pics\图解线程池实现原理.png)

1. 提交一个任务，线程池里存活的核心线程数小于线程数corePoolSize时，线程池会创建一个核心线程去处理提交的任务。
2. 如果线程池核心线程数已满，即线程数已经等于corePoolSize，一个新提交的任务，会被放进任务队列workQueue排队等待执行。
3. 当线程池里面存活的线程数已经等于corePoolSize了，并且任务队列workQueue也满，判断线程数是否达到maximumPoolSize，即最大线程数是否已满，如果没到达，创建一个非核心线程执行提交的任务。
4. 如果当前的线程数达到了maximumPoolSize，还有新的任务过来的话，直接采用拒绝策略处理。



### 2.3 线程池异常处理

如果被问到线程池异常处理，可以从以下角度回答：[#参考文章](https://juejin.cn/post/6844903889678893063)

- `try-catch` 捕获异常；
- submit执行的任务，可以通过 Future对象的get方法接收抛出的异常，然后再进行处理；
- 重写 `ThreadPoolExecutor.afterExcute` 方法，处理传递的异常；
- 设置`Thread.UncaughtExceptionHandler` 处理未检测的异常。



### 2.4 线程池的工作队列

（1）ArrayBlockingQueue（有界队列）

一个用数组实现的有界阻塞队列，按FIFO排序量

（2）LinkedBlockingQueue（可设置容量队列）

基于链表结构的阻塞队列，按FIFO排序任务，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX_VALUE，吞吐量通常要高于ArrayBlockingQuene；

**newFixedThreadPool线程池使用了这个队列**

（3）DelayQueue（延迟队列）

一个任务定时周期的延迟执行的队列。根据指定的执行时间从小到大排序，否则根据插入到队列的先后排序。

**newScheduledThreadPool线程池使用了这个队列**

（4）PriorityBlockingQueue（优先级队列）

一个具有优先级的无界阻塞队列。

（5）SynchronousQueue（同步队列）

一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene。

**newCachedThreadPool线程池使用了这个队列**



## 3、几种常见的线程池

- newFixedThreadPool (固定数目线程的线程池)

- newCachedThreadPool(可缓存线程的线程池)

- newSingleThreadExecutor(单线程的线程池)

- newScheduledThreadPool(定时及周期执行的线程池)

### 3.1 newFixedThreadPool

```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

线程池特点：

- 核心线程数和最大线程数大小一样
- 没有所谓的非空闲时间，即keepAliveTime为0
- 阻塞队列为无界队列LinkedBlockingQueue

工作机制：

- 提交任务
- 如果线程数少于核心线程，创建核心线程执行任务
- 如果线程数等于核心线程，把任务添加到LinkedBlockingQueue阻塞队列
- 如果线程执行完任务，去阻塞队列取任务，继续执行。

存在问题：**使用无界队列的线程池会导致内存飙升**，如果线程获取一个任务后，任务的执行时间比较长，会导致队列的任务越积越多，导致机器内存使用不停飙升， 最终导致OOM。

使用场景：FixedThreadPool适用于处理CPU密集型的任务，确保CPU在长期被工作线程使用的情况下，尽可能地少分配线程，即适用执行长期地任务。

### 3.2 newCachedThreadPool

```java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

线程池特点：

- 核心线程数为0
- 最大线程数为Integer.MAX_VALUE
- 阻塞队列是SynchronousQueue
- 非核心线程空闲存活时间为60秒

当提交任务的速度大于处理任务的速度时，每次提交一个任务，就必然会创建一个线程。极端情况下会创建过多的线程，耗尽 CPU 和内存资源。由于空闲 60 秒的线程会被终止，长时间保持空闲的 CachedThreadPool 不会占用任何资源。

工作机制：

- 提交任务
- 因为没有核心线程，所以任务直接加到SynchronousQueue队列
- 判断是否有空闲线程，如果有，就去取出任务执行
- 如果没有空闲线程，就新建一个线程执行
- 执行完任务的线程，还可以存活60秒，如果在这期间，接到任务，可以继续活下去；否则，被销毁。

使用场景：用于并发执行大量短期的小任务。

### 3.3 newSingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

线程池特点：

- 核心线程数为1
- 最大线程数也为1
- 阻塞队列是LinkedBlockingQueue
- keepAliveTime为0

工作机制：

- 提交任务
- 线程池是否有一条线程在，如果没有，新建线程执行任务
- 如果有，讲任务加到阻塞队列
- 当前的唯一线程，从队列取任务，执行完一个，再继续取，一个人（一条线程）夜以继日地干活。

使用场景：适用于串行执行任务的场景，一个任务一个任务地执行。

### 3.4 newScheduledThreadPool

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

线程池特点：

- 最大线程数为Integer.MAX_VALUE
- 阻塞队列是DelayedWorkQueue
- keepAliveTime为0
- scheduleAtFixedRate() ：按某种速率周期执行
- scheduleWithFixedDelay()：在某个延迟后执行

工作机制：

- 添加一个任务
- 线程池中的线程从 DelayQueue 中取任务
- 线程从 DelayQueue 中获取 time 大于等于当前时间的task
- 执行完后修改这个 task 的 time 为下次被执行的时间
- 这个 task 放回DelayQueue队列中

使用场景：周期性执行任务的场景，需要限制线程数量的场景。



## 4、线程池状态

![线程池状态](pics\线程池状态.png)

**RUNNING**

- 该状态的线程池会接收新任务，并处理阻塞队列中的任务;
- 调用线程池的shutdown()方法，可以切换到SHUTDOWN状态;
- 调用线程池的shutdownNow()方法，可以切换到STOP状态;

**SHUTDOWN**

- 该状态的线程池不会接收新任务，但会处理阻塞队列中的任务；
- 队列为空，并且线程池中执行的任务也为空,进入TIDYING状态;

**STOP**

- 该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，而且会中断正在运行的任务；
- 线程池中执行的任务为空,进入TIDYING状态;

**TIDYING**

- 该状态表明所有的任务已经运行终止，记录的任务数量为0。
- terminated()执行完毕，进入TERMINATED状态

**TERMINATED**

- 该状态表示线程池彻底终止

