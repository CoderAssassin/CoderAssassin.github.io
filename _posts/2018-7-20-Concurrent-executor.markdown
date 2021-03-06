---
layout:     post
title:      "并发任务执行"
subtitle:   " executor "
date:       2018-07-20 22:00:00
author:     "Aliyang"
header-img: "img/post-bg-concurrent-executor.jpg"
tags:
    - java并发
---
## 在线程中执行任务
1. 串行地执行任务
``` java
//例如：例如串行的Web服务器每次只处理一个请求
public class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    }
}
```

2. 显式地为任务创建线程
``` java
//例如：为每个请求创建一个新的线程来提供服务
public class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            //为每个任务创建一个线程
            new Thread(task).start();
        }
    }
}
```

3. 无限制创建线程的不足
	* 线程生命周期的开销非常高
	* 资源消耗
	* 稳定性：可创建线程的数量受到诸如JVM的启动参数、Thread构造函数中请求的栈大小以及底层操作系统对线程的限制等限制。

## Executor框架
在Java类库中，任务执行的主要抽象不是Thread，而是Executor：
``` java
public interface Executor{
    void execute(Runnable command);
}
```
Executor基于生产者-消费者模式，将任务提交与任务执行分开，提交任务相当于生产者，执行任务相当于消费者。

1. 示例：基于Executor的Web服务器。
``` java
//例如：基于线程池的Web服务器
public class TaskExecutionWebServer {
    private static final int NTHREADS = 100;
    private static final Executor exec
            = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            exec.execute(task);    //将任务提交到工作队列中
        }
    }
}
//为每个请求启动一个新线程的Executor：
public class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    };
}
//在调用线程中以同步方式执行所有任务的Executor：
public class WithinThreadExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();
    };
}
```

2. 执行策略
	* 在什么(What)线程中执行任务?
	* 任务按照什么(What)顺序执行(FIFO, LIFO、优先级)?
	* 有多少个(How Many )任务能并发执行?
	* 在队列中有多少个(How Many )任务在等待执行?
	* 如果系统由于过载而需要拒绝一个任务，那么应该选择哪一个(Which)任务?另外，如何(How)通知应用程序有任务被拒绝?
	* 在执行一个任务之前或之后，应该进行哪些(What )动作?

3. 线程池的主要参数
	* corePoolSize：核心线程池大小
	* maximumPoolSize：最大线程池大小
	* keepAliveTime：线程中线程数超过corePoolSize后空闲线程的最大存活时间；若设置allowCoreThreadTimeOut(true)那么线程数少于corePoolSize后也会有回收时间
	* TimeUnit：keepAliveTime时间单位
	* workQueue：阻塞队列
	* threadQueue：新建线程工厂
	* RejectedExecutionHandler：当提交任务数超过corePoolSize+maximumPoolSize和的时候，任务交给RejectedExecutionHandler来处理
4. 线程池
	* 指管理一组同构工作线程的资源池。
	* 线程池与工作队列密切相关，在工作队列中保存了所有等待执行的任务。
	* 可以通过调用Executors中的静态工厂方法之一来创建线程池：
		* newFixedThreadPool：创建固定长度线程池，每提交一个任务创建一个线程，直到最大数量。
			* 用一个无界队列LinkedBlockQueue存储提交的任务，因此多余的任务会一直存放再队列中，而不是创建新的线程。
			* corePoolSize=maximumPoolSize
		* newCachedThreadPool：创建可缓存的线程池，如果线程池的当前规模超过了处理需求时，那么将回收空闲的线程，而当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制。
			* 用一个无容量的SynchronizedQueue来保存任务，所以任务进来不会进队列，直接创建新的线程。
			* corePoolSize=0,maximumPoolSize=最大整数，keepAliveTime=60s，新的任务进来以后都是新创建一个线程，空闲线程达到60秒后直接回收
		* newSingleThreadExecutor：单线程的Executor，创建单个工作者线程来执行任务，如果这个线程异常结束，会创建另一个线程来替代。
			* corePoolSize=maximumPoolSize=1，用无界队列LinkedBlockingQueue来保存任务，所以每次都只有一个任务在执行。
		* newScheduledThreadPool：创建固定长度线程池，且以延迟或定时的方式执行任务，类似Timer。
			* 用无界延迟队列DelayedWorkQueue来保存任务
			* maximumPoolSize=最大整数，但是这个数没有意义，因为任务队列是无界队列，当线程数达到corePoolSize后会一直保持这个数，多的任务保存在队列里。

5. 线程池的运行流程
	* 当线程池的线程数小于corePoolSize时，新提交的任务都会创建一个新的线程来执行，即使线程池中现在有空闲的线程。
	* 当线程池的线程数达到corePoolSize后，新提交的任务会被放到阻塞队列中，等待线程池中任务调度进行。
	* 如果任务队列满了，并且maximumPoolSize>corePoolSize，那么就会继续创建新的线程来执行任务，直到线程数达到maximumPoolSize。
	* 当任务数超过maximumPoolSize后，那么新提交的任务由RejectedExecutionHandler处理。
	* 当线程池中线程数超过corePoolSize后，若空闲线程空闲时间达到keepAliveTime后，那么会关闭该空闲线程。
	* 如果设置allowCoreThreadTimeOut(true)，那么线程数小于corePoolSize时，空闲线程达到keepAliveTime也将关闭。

6. Executor的生命周期
	* Executor的实现通常会创建线程来执行任务，但JVM只有在所有(非守护)线程全部终止后才会退出，如果无法正确地关闭Executor，那么JVM将无法结束。
	* ExecutorService接口继承自Executor，添加了一些用于声明周期管理的方法。
		* 生命周期有3种状态：运行、关闭和已终止。
		* 关闭后提交的任务将由**“拒绝执行处理器”**处理，抛弃任务或者使得execute方法抛出一个未检查的**RejectedExecutionException**，等所有任务完成后，进入终止状态。
		* **awaitTermination**等待到达终止，**isTerminated**轮询**ExecutorService**是否已经终止。

7. 延迟任务与周期任务
	* 用ScheduledThreadPoolExecutor代替Timer，可以通过构造函数或者newScheduledThreadPool工厂方法来创建该类的对象。
		* Timer执行所有定时任务只能创建一个线程，如果某个任务的执行时间过长，那么将破坏其他TimerTask的定时精确性。
		* 如果TimerTask抛出一个未检查的异常，将表现出糟糕的行为。Timer线程并不捕获异常，因此当TimerTask抛出未检查的异常时将终止定时线程，会错误认为整个Timer都被取消，TimerTask将不会再执行，新的任务也不能调度。
	* DelayQueue
		* 实现了BlockingQueue，为ScheduledThreadPoolExecutor提供调度功能。每个对象有一个相应的延迟时间，逾期后才能从DelayQueue执行take操作。

## 找出可利用的并行性
Executor框架帮助指定执行策略，但如果要使用Executor，必须将任务表述为一个Runnable。
#### 携带结果的任务Callable与Future
* Runnable不能返回一个值或者一个受检查的异常。
* 对于存在延迟的计算——执行数据库查询等，Callable是一种更好的抽象：它认为主入口点(call)将返回一个值。
* Executor执行的任务有4个生命周期阶段:创建、提交、开始和完成。
* Future表示一个任务的生命周期，并提供了相应的方法来判断是否已经完成或取消，以及获取任务的结果和取消任务等。隐含意义是生命周期只能前进不能后退。
* 多种方法创建一个Future描述任务：
	* ExecutorService中的所有submit方法都将返回一个Future，从而将一个Runnable或Callable提交给Executor，并得到一个Future用来获得任务的执行结果或者取消任务。
	* 显式地为某个指定的Runnable或Callable实例化一个FutureTask。

``` java
//例如：使用Future实现一边下载图像一边渲染图像
public abstract class FutureRenderer {
    private final ExecutorService executor = Executors.newCachedThreadPool();
    void renderPage(CharSequence source) {
        final List<ImageInfo> imageInfos = scanForImageInfo(source);
        Callable<List<ImageData>> task =
                new Callable<List<ImageData>>() {
                    public List<ImageData> call() {
                        List<ImageData> result = new ArrayList<ImageData>();
                        for (ImageInfo imageInfo : imageInfos)
                            result.add(imageInfo.downloadImage());
                        return result;
                    }
                };

        Future<List<ImageData>> future = executor.submit(task);//将Runnable或者Callable提交给ExecutorService，然后返回一个Future对象
        renderText(source);

        try {
            List<ImageData> imageData = future.get();
            for (ImageData data : imageData)
                renderImage(data);
        } catch (InterruptedException e) {
            // Re-assert the thread's interrupted status
            Thread.currentThread().interrupt();
            // We don't need the result, so cancel the task too
            future.cancel(true);
        } catch (ExecutionException e) {
            throw launderThrowable(e.getCause());
        }
    }
}
```
#### CompletionService:Executor与BlockingQueue
1. CompletionService
Future的get会轮询判断任务是否完成，有些繁琐，更好的使用方法是CompletionService。
CompletionService将Executor和BlockingQueue的功能融合在一起。你可以将Callable任务提交给它来执行，然后使用类似于队列操作的take和poll等方法来获得已完成的结果，而这些结果会在完成时将被封装为Future。

2. ExecutorCompletionService
	* 在构造函数中创建一个BlockingQueue来保存计算完成的结果。
	* 提交的任务首先包装成一个QueueingFuture，FutureTask的子类，改写done方法，将结果放在BlockingQueue中，take和poll方法委托给了BlockingQueue，会在得出结果前阻塞(其实就是提交的任务包装成QueueingFuture，从这个对象获得结果保存到BlockingQueue中，take和poll都是对BlockingQueue的)。

3. 为任务设置时限
	* 如果某个任务无法在指定时间内完成，那么将不再需要它的结果。
	* 在支持时间限制的Future.get中支持这种需求:当结果可用时，它将立即返回，如果在指定时限内没有计算出结果，那么将抛出TimeoutException。
