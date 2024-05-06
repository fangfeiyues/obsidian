
## 1、线程

### 1.1 线程创建&运行

-  **创建**

	1.  继承 Thread 类创建线程
	2.  实现 Runnable 接口创建线程
	3.  通过 Callable 和 FutureTask 创建线程 ？？
	4.  通过 线程池 创建


-  **等待&唤醒**

	1.  wait vs sleep
		1.  sleep 可以在任何地方使用；wait 只能在同步方法或同步块使用
		2.  wait 会释放锁，sleep 不会
	2. run vs start


-  **线程数设定**

	CPU密集型（计算占主导，增加线程不会提升性能，因为大多数时间都在CPU计算）则 设置 N + 1
	IO密集型（等待占主导，可以分配更多的线程，来提升CPU的利用率） 则 设置 2N + 1


### 1.2 线程并发&安全

`解决多线程间 并发 访问共享变量问题，以使得程序正确完成

-  **线程同步**

	让多个线程按照顺序访问同一个资源，避免因并发导致的冲突问题，主要有以下几种方式：
	1.  synchronized：基本的线程同步机制
	2.  reentrantLock：可支持公平锁、可中断锁、多个条件变量等功能
	3.  semaphore：允许多个线程同时访问共享资源，但限制访问线程数量
	4.  countDownLatch：允许一个或多个线程等待其他线程执行完毕后再执行，可用于线程间协调与通信
	5.  cyclicBarrier：允许多个线程在一个栅栏处等待，直到所有线程都到达栅栏


-  **死锁**

	产生死锁的4个必要条件
	1.  互斥条件
	2.  占有且等待
	3.  不可强行占有
	4.  循环等待条件


### 1.3 线程等待&通知

`解决线程之间等待与通知问题

- **volatile vs synchronized**

	关键字volatile可以用来修饰字段，告诉程序任何对变量的访问均需要从共享内存获取而对他的改变必须同步刷新到共享内存。从而保证所有线程对变量访问的可见性。
	
	关键字synchronized可以确保多个线程在同一时刻只有一个线程处于方法或同步块中，保证了可变和排他


- **等待/通知机制**

	等待方执行原则如下
	1.  获取对象的锁
	2.  如果条件不满足那么调用对象的wait()方法，被通知后仍要检查条件
	3.  条件满足则执行对应的逻辑
	
	通知方遵守原则亦是


- **ThreadLocal**

	解决多线程并发问题，通过为每个线程创建一份共享变量来保证线程之间的变量访问和通信互相不影响



## 2、多线程

### 2.1 上下文切换

-  **含义**

	上下文切换通常是指在一个CPU上，由于多线程共享一个CPU时间片，当一个线程的时间片用完后，需要切换到另一个线程运行。此时需要保存当前线程的状态信息，包括程序计数器、寄存器、栈指针等，以便下次继续执行该线程时能恢复到正确的执行状态

- **减少切换**

	由于多线程切换需要保存和恢复更多的上下文信息，这会降低系统的运行效率，所以要尽量减少
	1.  减少线程数，通过合理的线程池来管理线程
	2.  使用无锁并发编程，无锁并发编程可以避免线程因等待而进入阻塞状态，从而减少上下文的却换
	3.  使用CAS算法，避免线程阻塞和唤醒
	4.  使用协程

### 2.2 线程池

#### 参数

```java
// 
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

-  **corePoolSize**

	核心线程数量，可以类比正式员工数量

-  **maximumPoolSize**

	最大线程数量，常驻+临时线程

-  **workQueue**

	多余任务等待队列，再多的人都处理不过来了，需要等着，在这个地方等

-  **keepAliveTime**

	非核心线程空闲时间

-  **threadFactory**

	创建线程的工厂

-  **handler**

	拒绝策略


#### 实现

池化，提前创建好一批线程，然后保存在线程池中，当有任务需要执行的时候，从线程池中选一个线程来执行

所谓线程池本质是一个 hashSet 来存放 worker，多余的任务会放在阻塞队列中。

只有当阻塞队列满了后，才会触发非核心线程的创建，所以非核心线程只是临时过来打杂，直到空闲了，然后自己关闭了

线程池提供了两个钩子 beforeExecute，afterExecute 给我们继承，在执行任务前后做一些事

线程池原理关键技术：锁（lock，cas）、阻塞队列、hashSet（资源池）









