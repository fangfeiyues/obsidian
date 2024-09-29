## 1、synchronized

### 原理

	synchronized 锁的是个什么？是怎么实现的锁

- **实现**

	1、方法级别的
	
	方法常量池会有一个 `ACC_SYNCHRONIZED` 标志，当某个线程要访问某个方法时，会检查这个标志，如果有设置，则需先获得监视器锁，然后开始执行方法，并在执行后释放
	
	2、代码级别的
	
	使用 `monitor_enter` 和 `monit_exit` 实现，可以理解为一个加锁一个释放锁。每个对象维护着一个记录着被锁次数的计数器，未被锁定对象的该计数器为0，当一个线程获得锁后自增为1，释放自减，当计数器为0的时候，锁将被释放，其他线程可以获得锁


-  **Monitor**

	为了解决线程安全问题，Java提供了同步机制、互斥锁机制，这个机制保证了在同一时刻只有一个线程能访问共享资源。这个机制的保障来源于监视锁 Monitor，其代码由C++实现
	
	![[image-Concurrent-03 并发锁-20240630162556819.png]]
	
	其过程大致是：一个对象想要获取锁 --> 先进入 `Entry Set` 等待区间 --> Monitor基于一个标准如FIFO进入房间 --> 房间内线程释放或挂起线程，则进入 `Wait Set` 等待队列 --> 


-  **锁消耗**

	1. 性能消耗：加锁、解锁过程中的损耗
	   
	2.  产生阻塞：基于Monitor对象，当对线程同时访问一段同步代码时，首先会进入Entry Set，当一个线程获得对象的锁后，才能进入The Owner 区域，其他线程还在继续等待


### 使用

-  **vs Lock**

	lock 与 sychronized 都能保证线程的原子性、可见性、一致性，但 lock 比 sychronied 更灵活
	1. lock 是 jdk 层面上的实现，sychronized是jvm 层面的关键字
	2. b.lock.lock()与lock.unlock()可以主动获取锁释放锁，sychronized是被动的释放锁，sychronized 释放锁的时机：i、代码执行执行 ii、产生异常了
	3. lock可以判断锁的状态，sychronized 是一个关键字，没法去判断锁的状态
	4. lock可以指定公平锁和非公平锁，sychronized只能作为非公平锁
	5. lock可以有共享锁与排他锁，例如读写锁的实现。而sychronized是一个可重入的排他锁


-  **原子性**

	通过 monitor_enter 和 monitor_exist 字节码指令，当线程执行到 monitor_enter 的时候要先获得锁，才能执行后面方法，当线程执行到 monitor_exist 的时候则要释放锁。这时候其他线程则无法获取锁，保证方法和代码块内的原子性


- **有序性**

	synchronized 并不能直接解决有序性问题，还是通过 as-if-serial（不管怎么重排序单线程执行顺序结果都不能被改变） +  synchronized（保证同一时间只能被同一线程执行） = 有序性


- **可见性**

	对于一个变量解锁之前，必须先把此变量同步回主存。这样解锁后，后续线程可与你访问到被修改的值

### 锁升级

```
Q: 为什么会锁升级？
A: JDK 1.6 及之前版本，如果对象被锁线程就会进入阻塞知道锁被释放，因为锁的获取和释放都需要在操作系统层面进行线程的阻塞和唤醒，而这些操作会带来很大的开销（用户态->内核态）。这时如果通过轻量锁自旋等待方式，会减少不少
```

-  **偏向锁**

	1. 触发条件：当一个线程首次进入时，锁对象会进入偏向模式
	
	2. 实现方式：JVM会在对象头记录该线程ID作为偏向锁持有者，并将对象头中的 Mark Word 中的一部分作为偏向锁标识 01
	
	3. 结果：这种情况下其他线程访问该对象，会检查对象的偏向锁标识

- **轻量级锁**

	1. 触发条件：当有另一个线程尝试获取已被偏向的锁时，偏向锁会被撤销，锁会升级为轻量级锁
	
	2. 实现方式：
		1. 当一个线程尝试获取轻量级锁时，JVM会在当前线程栈帧中创建一个锁记录空间，然后将对象头中的Mark Word复制到这个锁记录（复制是为了在释放的时候恢复Mark Word对象头）
		2.  JVM 使用 CAS(Compare-And-swap) 将对象头的 Mark Word 更新为 指向锁记录的指针，如果失败默认自旋15次，失败后升级重量级锁

- **重量级锁**

	1. 触发条件：当轻量级锁的CAS操作失败，锁会进一步升级为重量级锁
	
	2. 实现方式：当锁状态升级到重量级锁状态时，JVM会将该对象的锁变成一个重量级锁，并在对象头中记录指向等待队列的指针。
	
	3. 结果：此时如果一个线程想获取该对象的锁，则需要先进入等待队列，等待该锁被释放。当锁被释放时，JVM会从等待队列中选择一个线程唤醒，并将该线程的状态设置为就绪，然后等待该线程重新获取对象锁（这时候直接阻塞而不考虑重量级锁的自旋？一般时间比较长，短时间可以考虑）


-  **自适应自旋**

	- JDK6后引入自适应自旋机制，通过监控轻量级锁等待情况，动态调整自旋等待时间，如果时间很短说明锁的竞争不激烈，可以线程自旋等待避免线程挂起和恢复带来的性能损失；如果等待时间长，则说明竞争激烈，应该及时释放CPU资源
	  
	- 自旋实现用C++ 的 inline函数

## 2、volatile

为什么总感觉实际开发中很少使用volatile？

### 可见性

当对volatile变量进行写操作时，JVM会向处理器发送一条lock指令，将缓存中的变量回写到系统主存中

### 有序性

volatile通过内存屏障来禁止指令重排，这就保证了代码程序回严格按照代码的先后顺序执行。

![[image-Concurrent-03 并发锁-20240504154404478.png|600]]

Thread_2 在判断 singleton != null 后直接返回就可能造成NPE，因为 Thread_2 拿到并不是一个完整对象
[[JVM-02 对象初始化与加载#2、对象加载过程]]  这个过程并不是一个原子操作并且编译器可能重排序先把内存地址赋予变量再初始化，那么使用volatile即可解决问题

## 3、aqs

### 3.1 原理

AQS（AbstractQueuedSynchronizer）是为了解决多个线程同时抢占一个或多个资源时出现的并发问题
```text
抽象同步队列器是实现锁的关键，在锁的实现中聚合同步器利用同步器实现锁的语义。

可以理解二者之间的关系：
	1. 锁是面向使用者的它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；
	2. 同步器面对的是锁的使用者它简化了锁的实现方式，屏蔽了同步状态管理线程的排队等待与唤醒等底层操作。
锁和同步器很好的隔离了使用者和实现者锁需要关注的领域

```

AQS内部，维护了 一个FIFO队列 和 一个 volatile 的 state 变量，在 state = 1 的时候表示当前对象锁已经被占有，state 的修改动作通过CAS完成

#### FIFO队列

FIFO队列来实现多线程的排队工作，当线程加锁失败时，该线程会被封装成一个Node节点置于队尾

![[image-Concurrent-03 并发锁-20240504162631647.png|600]]

当持有锁的线程释放锁时，AQS会将等待队列中的第一个线程唤醒，并让其重新尝试获取锁

![[image-Concurrent-03 并发锁-20240504162726721.png|500]]

FIFO的简单流程

![[image-Concurrent-03 锁结构-20240429020505420.png|500]]

![[image-Concurrent-03 锁结构-20240429020524289.png|500]]

`waitStatus = SIGNAL（值为-1）`，后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或被取消将会通知后续节点，使得后续节点的线程得以运行
`waitStatus = CANCELLED（值为1）`，由于在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待节点进入该状态将不再变化


-  **条件队列**

	同步队列主要实现锁机制，条件队列用于线程间的通信和协调。比较典型的有 ReentrantLock 的 Condition

#### State状态

volatile 修饰变量，state = 1 表示当前对象锁已被占用

#### CAS

-  **应用**

	Compare And Swap 在进行并发修改时，会先比较预期和取出的值是否相等，相等则会把值替换成新值
	CAS主要应用的就是乐观锁和锁自旋

-  **原理**

	Unsafe是CAS核心类，因为Java无法直接访问底层操作系统，而是通过本地native方法来访问，不过尽管如此，JVM还是开了后门Unsafe类来提供原子级别操作
	
	CAS在x86架构的CPU中，通常使用 cmpxchg 指令实现，这个指令可以保证原子性：
	1、cmpxchg 是一条原子指令，在CPU执行指令的时候处理器会自动锁定总线，防止其他CPU访问共享变量，然后执行比较和交换操作，最后释放总线
	2、cmpxchg 执行期间，会自动禁止中断
	3、cmpxchg 指令是硬件实现的，可以保证其原子性与正确性

-  **CAS + Volatile**

	CAS可以保证原子性，即一个修改命令以原子性操作完成不会中断，但并发编程中还需要可见性和有序性，即一个线程通过CAS改变一个变量的值后，还需要被其他线程看见


-  **ABA问题**

	1.  Thread_1 -> A
	2.  Thread_2 -> A
	3.  Thread_2 -> A to B
	4.  Thread_2 -> B to A
	5.  Thread_1 -> CAS success.
	
	解决方案：版本号



#### 等待与唤醒

AQS中的等待与唤醒主要依赖 park 和 unpark 实现，当一个线程尝试获取失败，AQS会将该线程封装成一个Node并添加到等待队列，然后通过 LockSupport.park() 将该线程阻塞

此外AQS还提供了条件变量 Condition实现 ？？？


### 3.2 模式
#### 独占模式

独占模式意味着一次只有一个线程可以获取同步状态，通常用于互斥如 ReentrantLock（读写锁）

**acquire阻塞获取独占锁**

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```


**release释放同步状态**

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```


**tryAcquireNanos独占式超时获取同步状态**

提供了传统的synchronized所不具备的特性。

![[image-Concurrent-03 锁结构-20240429020608359.png|600]]






### 3.3 公平与非公平

-  **公平锁**

	多个线程按照申请锁的顺序去获取锁，保证队列第一个先得到
	
	优点是所有线程都能得到资源不会饿死在队列，但他存在吞吐量下降，除了第一个线程其他都会阻塞，CPU唤醒阻塞开销也会很大缺点

-  **非公平**

	多个线程不按申请顺序去获取锁，而是直接尝试，获取不到再进入等待队列。如 ReentrantLock默认非公平
	
	优点是减少开销，吞吐率会高不少，但可能会一直等待锁取不到



### 3.4 实现类

-  **ReentrantLock**

```java
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
			// 是否能重入标识
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```


	nonfairTryAcquire 新加 hasQueuedPredecessors 即加入了同步队列中当前节点是否有前驱节点的判断
	
	非公平可能使得线程发生饥饿的情况，为什么还被设置为默认？极少的线程上下文切换保证了吞吐量


-  **CountDownLatch**

	计数器，允许一个或多个线程等待其他线程完成操作后，再继续执行


-  **CyclicBarrier**

	同步屏障，允许多个线程等待直到某个公共屏障点，才能继续执行


-  **Semaphore**

	计数信号量，允许多个线程同时访问共享资源，并通过计数器控制访问数量