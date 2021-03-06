---
layout:     post
title:      "并发基础构建模块"
subtitle:   " concurrent "
date:       2018-07-20 16:30:00
author:     "Aliyang"
header-img: "img/post-bg-concurrent-basemodel.jpg"
tags:
    - java并发
---

## 同步容器类
如Vector和Hashtable，以及JDK1.2添加的功能类似的类，都是由Collections.synchronizedXxx等工厂方法创建的。
实现线程安全的方式：将它们的状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。

#### 同步容器类的问题
1. 同步容器类都是线程安全的，但在某些情况下可能需要额外的客户端加锁来保护复合操作，如迭代、跳转以及条件运算。在同步容器类中，这些复合操作在没有客户端加锁的情况下仍然是线程安全的，但当其他线程并发地改容器时，它们可能会表现出意料之外的行为。
```
public static Object getLast(Vector list){
    int lastIndex=list.size()-1;
    return list.get(lastIndex);
}
public static void deleteLast(Vector list){
    int lastIndex=list.size()-1;
    list.remove(lastIndex);
}
```
当A在一个Vector上调用getLast，B在同一个Vector上调用deleteLast，会报异常：![1](https://github.com/CoderAssassin/markdownImg/blob/master/JavaConcurrent/1.png?raw=true)

2. 由于同步容器类支持客户端加锁，可能会创建一些新的操作，只要我们知道应该使用哪一个锁，那么这些操作就与容器的其他操作一样都是原子操作。
```
public static Object getLast(Vector list){
    synchronized(list){//对list对象加锁
        int lastIndex=list.size()-1;
        return list.get(lastIndex);
    }
}确保
public static void deleteLast(Vector list){
    synchronized(list){
        int lastIndex=list.size()-1;
        list.remove(lastIndex);
    }
}
```
上述代码将getLast和deleteLast都变成原子性操作，可以保证size和get之间不会发生变化。

3. 在调用size和get之间，Vector的长度仍然可能会发生变化，这种风险在对Vector中的元素迭代时仍然会出现(这里假设在迭代的过程中删除元素，那么size()会跟着变化，get(i)得到的值并不是预期的值)，可以在迭代期间持有Vector锁，但会牺牲一定的伸缩性。
``` java
//下面代码当迭代过程中删除对象时，会出现ArrayIndexOutOfBoundsException异常。
for(int i=0;i<vector.size();i++)
	doSomething(vector.get(i));
//下面对整个vector加锁，防止结构被修改:
synchronized(vector){
    for(int i=0;i<vector.size();i++)
        doSomething(vector.get(i));
}
```

#### 迭代器与ConcurrentModificationException(fail-fast)
容器类在迭代的时候当发现容器在迭代的过程中被修改时，会抛出ConcurrentModificationException异常。
避免方式：

* 一种是在迭代期间对容器加锁，但是若迭代周期很长，其他线程将会等待很长时间。
* 另外一种方法是“克隆”容器，在线程副本上迭代，因为副本是封闭在线程内的，其他线程不会在迭代期间对其进行修改。

#### 隐藏迭代器
```
public class HiddenIterator {
    @GuardedBy("this") private final Set<Integer> set = new HashSet<Integer>();

    public synchronized void add(Integer i) {
        set.add(i);
    }

    public synchronized void remove(Integer i) {
        set.remove(i);
    }

    public void addTenThings() {
        Random r = new Random();
        for (int i = 0; i < 10; i++)
            add(r.nextInt());
        System.out.println("DEBUG: added ten elements to " + set);
    }
}
```
编译器将字符串的连接操作转换为调用StringBuilder.append(Object)，而这个方法又会调用容器的toString方法，标准容器的toString方法将迭代容器，并在每个元素上调用toString来生成容器内容的格式化表示，根本问题在于set进行打印输出之前没有获取HiddenIterator的锁。
如果用synchronizedSet包装HashSet，并且对同步代码进行封装，可以避免这种错误。

## 并发容器
同步容器将所有对容器状态的访问都串行化，以实现他们的线程安全性，代价是严重降低并发性。通过使用并发容器来代替同步容器，可以极大地提高伸缩性并降低风险。

#### ConcurrentHashMap
* 同步容器类在执行每个操作期间都持有一个锁，例如HashMap.get或List.contains可能包含大量的equals等工作，会花费很长时间，其他线程在此期间没法访问容器
* 分段锁：ConcurrentHashMap使用一种称为分段锁的机制，在这种机制中，任意数量的读取线程可以并发地访问Map，执行读取操作的线程和执行写入操作的线程可以并发地访问Map，并且一定数量的写入线程可以并发地修改Map。
* 只有当应用程序需要加锁Map以进行独占访问时，才应该放弃使用ConcurrentHashMap，如size(size返回的结果实际上只是一个估计值，因为返回的结果在计算时可能已经过期)和isEmpty。
具体的在我的另外一个博客中有详解：[https://coderassassin.github.io/2018/07/16/java-ConcurrentHashMap/](https://coderassassin.github.io/2018/07/16/java-ConcurrentHashMap/)

#### CopyOnWriteArrayList
* 用于替代同步List，在迭代期间不需要对容器进行加锁或复制
* “写入时复制”安全性：通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。
	
## 阻塞队列和生产者-消费模式

#### 阻塞队列
BlockingQueue的put和take方法会引起阻塞，使用阻塞机制可以实现将BlockingQueue作为池，put为生产，take为消费，简单的"生产者-消费者"模式。
BlockingQueue有多种实现，LinkedBlockingQueue和ArrayBlockingQueue，还有PriorityBlockingQueue。

#### 串行线程封闭
* 线程封闭对象只能由单个线程拥有，可以通过安全地发布对象来转移所有权，这样另一个线程获得访问权，发布的线程不会再访问。
* 对象池利用了串行线程封闭，将对象“借给”一个请求线程。只要对象池包含足够的内部同步来安全地发布池中的对象，并且只要客户代码本身不会发布池中的对象，或者在将对象返回给对象池后就不再使用它，那么就可以安全地在线程之间传递所有权。

#### 双端队列与工作密取
* Deque和BlockingDeque，具体实现由ArrayDeque和LinkedBlockingDeque。
* 双端队列适用于工作密取，在工作密取中，每个消费者都有各自的双端队列。如果一个消费者完成了自己双端队列中的全部工作，那么它可以从其他消费者双端队列末尾秘密地获取工作。适用于既是消费者也是生产者的问题。

## 阻断方法与中断方法
* 中断是一种协作机制。一个线程不能强制其他线程停止正在执行的操作而去执行其他的操作。当线程A中断B时，A仅仅是要求B在执行到某个可以暂停的地方停止正在执行的操作——前提是如果线程B愿意停止下来。
* 当某方法抛出**InterruptedException**时，表示该方法是一个阻塞方法。
* 当在代码中调用了一个将抛出InterruptedException异常的方法时，你自己的方法也就变成了一个阻塞方法，并且必须要处理对中断的响应。对于库代码来说，有两种基本选择:
	* 传递InterruptedException。只需把InterruptedException传递给方法的调用者。
	* 恢复中断。例如代码是Runnable的一部分时不能抛出异常，需要捕获异常然后用interrupt恢复，使得更高层的调用栈代码看到中断。

## 同步工具类
* 同步工具类可以是任何一个对象，只要它根据其自身的状态来协调线程的控制流。例如阻塞队列、信号量、栅栏和闭锁等。
* 所有的同步工具类都包含一些特定的结构化属性：它们封装了一些状态，这些状态将决定执行同步工具类的线程是继续执行还是等待，此外还提供了一些方法对状态进行操作，以及另一些方法用于高效地等待同步工具类进入到预期状态。

#### 闭锁
* 可以延迟线程的进度直到其到达终止状态。
* 闭锁可以用来确保某些活动直到其他活动都完成后才执行，如：
	* 确保某个计算在其需要的所有资源都被初始化之后才继续执行。
	* 确保某个服务在其依赖的所有其他服务都已经启动之后才启动。
	* 等待直到某个操作的所有参与者(例如，在多玩家游戏中的所有玩家)都就绪再继续执行。在这种情况中，当所有玩家都准备就绪时，闭锁将到达结束状态。
* CountDownLatch
	* 是一种灵活的闭锁实现，可以使一个或多个线程等待一组事件的发生。
	* 包括一个计数器，**countDown**方法递减计数器，表示一个事件已经发生，**await**方法等待计数器达到零，表示所有需要等待的事件都已经发生。
```
例如：在计时测试中使用CountDownLatch来启动和停止线程
public class TestHarness {
    public long timeTasks(int nThreads, final Runnable task)
            throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException ignored) {
                    }
                }
            };
            t.start();
        }

        long start = System.nanoTime();
        startGate.countDown();    //执行完这句后所有线程结束等待，然后开始执行task.run()和endGate.countDown()
        endGate.await();    //线程执行的时候等待所有任务完成
        long end = System.nanoTime();
        return end - start;
    }
}
```
用了两个闭锁，起始门计数器初始值为1，结束门计数器初始值为工作线程的数量。每个工作线程首先要做的事就是在启动门上等待，从而确保所有线程都就绪后才开始执行。而每个线程要做的最后一件事情是将调用结束门的countDown方法减1，这能使主线程高效地等待直到所有工作线程都执行完成，因此可以统计所消耗的时间。

#### FutureTask
* FutureTask也可以用作闭锁。FutureTask表示的计算是通过Callable来实现的，相当于一种可生成结果的Runnable，并且可以处于以下3种状态：
	* 等待运行
	* 正在运行
	* 运行完成：表示计算的所有可能结束方式，当FutureTask进入完成状态后会一直停在这个状态。
* Future.get的行为取决于任务的状态。如果任务已经完成，那么get会立即返回结果，否则get将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。
* FutureTask将计算结果从执行计算的线程传递到获取这个结果的线程，而FutureTask的规范确保了这种传递过程能实现结果的安全发布。
* Callable表示的任务可以抛出受检查的或未受检查的异常，并且任何代码都可能抛出一个**Error**。无论任务代码抛出什么异常，都会被封装到一个**ExecutionException**中，并在Future.get中被重新抛出。

#### 信号量
* 计数信号量用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。
* **Semaphore**中管理着一组虚拟的许可，许可的初始数量可通过构造函数来指定，在执行操作时可以首先获得许可(只要还有剩余的许可)，并在使用以后释放许可。如果没有，**acquire**将阻塞直到有许可，**release**方法将返回一个许可给信号量。

#### 栅栏
* 闭锁是一次性对象，一旦终止不会被重置。与闭锁的关键区别：所有线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待事件，而栅栏用于等待其他线程。
* **CyclicBarrier**可以使一定数量的参与方反复地在栅栏位置汇集。
	* 当线程到达栅栏位置时将调用**await**方法，该方法阻塞直到所有线程到达栅栏位置；若所有的线程到达栅栏的位置，栅栏打开，释放所有线程，重置栅栏。
	* 如果对await的调用超时，或者await阻塞的线程被中断，那么栅栏就被认为是打破了，所有阻塞的await调用都将终止并抛出**BrokenBarrierException**。
* Exchanger是一种两方栅栏，各方在栅栏位置上交换数据。
	* 当双方执行不对称的操作时，会非常有用。例如一个线程向缓冲区写入数据，另一个线程从缓冲区读取数据。
