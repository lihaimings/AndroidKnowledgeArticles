## 线程池
线程的除了执行任务时间，还需要创建、销毁、切换的时间，所以无限的线程创建销毁会造成资源无意义浪费，线程池就可以限定线程数量，并且可以设置核心线程数在执行完成后不销毁。

## ThreadPoolExecutor概述
`ThreadPoolExecutor`是继承`AbstractExecutorService->ExecutorService->Executor`。其中Executor、ExecutorService都是接口类。
**Executor:**定义了一个`execute(Runnable)`接口方法用于提交没有返回值的任务。
**ExecutorService:**则定义了许多接口方法，主要包括`submit()`提交任务并返回结果，和关闭线程池的`shutdown()`等。
![image.png](https://upload-images.jianshu.io/upload_images/22650779-2c7ee7cab183388d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/900)
所以定义线程池都是ExecutorService接口以上，sdk中提供了**`Executors`**用于创建一个简单的线程池如：
- Executors.newCachedThreadPool (创建无界线程池)
- Executors.newFixedThreadPool(int nThreads) (创建固定大小的线程池)
- Executors. newSingleThreadExecutor (创建单一线程池)


**ThreadPoolExecutor的构造方法：**
ThreadPoolExecutor必须要设置**核心数、线程最大数、线程存活时间、阻塞队列**，线程工厂和拒绝策略，可以不设置，也可以自定义。

![image.png](https://upload-images.jianshu.io/upload_images/22650779-c1351e2cdf69a4af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 核心数与最大线程数
- corePoolSize: 线程池的核心线程数
- maximumPoolSize：线程池的最大线程数
1. 如果线程池任务数<=corePoolSize核心数，则在线程池创建新线程执行任务，并且这些线程执行完任务后并不会销毁，而是执行在阻塞队列等待的任务或新任务。

2. 如果线程池任务数开始>corePoolSize核心数,则将任务送进阻塞队列中，直到阻塞队列满为止。

3.  如果线程池提交的任务数> corePoolSize核心数并且阻塞队列已满，但创建线程数< maximumPoolSize，则创建新的线程执行新任务，不过这些线程非核心线程，会被销毁。

**提前加载核心线程：**
如果我们想提前加载多一个核心线程数，那么可以调用`prestartCoreThread`方法，如果我们想提前加载完全部核心线程数则调用`prestartAllCoreThreads`方法。

![image.png](https://upload-images.jianshu.io/upload_images/22650779-62dad0bf09114228.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 线程存活时间
- keepAliveTime ： 存活时间的大小
- TimeUnit ： 时间的单位
非核心线程是会被销毁的，如果非核心线程处于空闲状态超过了我们指定的存活时间，就会被销毁。

#### Queuing队列
阻塞队列用于存放提交的任务，由上面的核心数和最大线程数介绍可知，阻塞队列会临时存储任务，在线程池空闲的情况下再从阻塞队列中拿出执行。

**1. 直接握手队列**
如`SynchronousQueue`它只起到一个传递作用，本身并不存储元素，就像传球一样，所以每一个put操作必须等待一个take操作。一般要有多余的maximumPoolSizes线程进行接应。

**2. 无界队列**
无界阻塞队列会使maximumPoolSizes失去意义，线程池中一直运行的是核心线程，这样能保证每一个请求都能得到响应，但是执行的效率可能会变慢。如`PriorityBlockingQueue、DelayQueue`就是无界阻塞队列


**3. 有界队列**
有界的队列和有限的maximumPoolSizes可以防止资源被耗尽，但是对于大量的请求，如果队列和最大线程都满的话，多余的任务可能会被丢弃。有界队列有：`ArrayBlockingQueue、LinkedBlockingQueue`

在自定义线程池中，我们必须要为线程池设置一个阻塞队列。

#### Reject Tasks 拒绝任务
当线程数已经达到最大数，并且阻塞队列已经满时，提交的任务就会交给`RejectedExecutionHandler`处理，比如是直接报错处理还是直接对接新的任务等。
ThreadPoolExecutor提供了四种拒绝处理：
- ThreadPoolExecutor.CallerRunsPolicy: 这提供了一个简单的反馈控制机制，可以减慢提交新任务的速度；

- ThreadPoolExecutor.AbortPolicy: 默认的策略，直接抛出RejectedExecutionException运行时异常

- ThreadPoolExecutor.DiscardPolicy：直接丢弃任务

- ThreadPoolExecutor.DiscardOldestPolicy 丢弃阻塞队列最前面的任务，然后重新尝试。


#### ThreadFactory线程工厂
ThreadFactory是为线程池中创建新线程，通过继承ThreadFactory可以为新线程定义如线程名字，优先级等设置。如果不设置，则默认使用Executors.DefaultThreadFactory的线程工厂。

## 线程池关闭
**线程的关闭**
线程的执行时，如果直接关闭可能导致线程无法释放锁，所以之前的`Thread.stop、destroy、suspend、resume`已经是过时方法。
线程提供`interrupt()`方法把线程的中断标志位设为`true`,但并不会自动的去结束线程：
1. 如果线程处于执行状态，则有线程`run`方法内部通过isInterrupted()方法来判断是否做线程结束操作，如果没有做相应的操作，那么即使执行interrupt()方法，对线程相当于不起作用。

2. 如果线程处于阻塞状态，那么线程会退出阻塞，并抛出**InterruptedException**异常，阻塞的线程可以通过捕获异常来做一些操作。

所以调用线程的`interrupt()`方法，能否关闭线程完全靠线程是否对中断标志有作处理。

**线程池关闭**
线程池自动关闭条件：**1. 程序对线程池没引用 ， 2.线程池中没有线程**
线程池关闭提供了两个方法：`shutdownNow、shutdown`
**shutdown:**拒绝线程池接收新的任务，它并没有对线程执行`interrupt()`方法。
**shutdownNow:**拒绝线程池接收新的任务，并对线程池中所有的线程调用`interrupt()`,那么所有阻塞线程将直接退出执行，并作为返回值。


## 线程池的合理配置
线程池的线程数应该配置多少才合理？，一个设配能执行多少个线程，决定于CPU的核心数，这个核心数可以通过代码拿到:
> Runtime.getRuntime().availableProcessors()

- 如果任务是计算密集型，则CPU的利用率很高，那么我们就把线程池的最大配置为CPU的核心数，那样就减少线程间的上下文切换。

- 如果任务是IO密集型，请求时会有大量的时间阻塞等待返回结果，那么我们可以尽量设置多的线程，进行线程切换执行其他任务。合理配置为：CPU核心数 * 2

## 线程池总结：
- 自定义线程池ThreadPoolExecutor比设置：核心数和最大线程数、非核心线程存活时间、阻塞队列。
阻塞队列有三种类型：`传递型、有界、无界`。阻塞队列的设置要提前判断好业务类型，因为有可能请求得不到相应，也可能无限制的请求。
线程工厂和拒绝策略：非必要设置，拒绝策略中,ThreadPoolExecutor提供了四种拒绝处理：`AbortPolicy:`直接抛异常，`DiscardPolicy:`直接丢弃处理，`DiscardOldestPolicy:`丢弃队头任务，`CallerRunsPolicy:`减慢提交速度。

- 线程池的关闭有`shutdownNow、shutdown`两种方法，它们都使得线程池不接收新任务，并且shutdownNow还会调用每个线程的`interrupt()`方法。

- 线程数的配置：如果任务是计算密集型，则减少上下文切换，设置为CPU的核心数。如果任务是IO密集型，则尽量切换其他线程，减少等待时，设置为CPU的核心数的2倍



