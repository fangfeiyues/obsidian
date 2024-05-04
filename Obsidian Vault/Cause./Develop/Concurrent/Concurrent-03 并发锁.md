## 1、Synchronized

### 使用

-  **阻塞性**

-  **互斥性**

-  **可重入性**

### 原理

- **实现**

	同步代码使用 monitor_enter 和 monit_exit 实现，每个对象维护着一个记录着被锁次数的计数器，未被锁定对象的该计数器为0，当一个线程获得锁后自增为1，释放自减，当计数器为0的时候，锁将被释放，其他线程可以获得锁

-  **锁消耗**

	1. 性能消耗：加锁、解锁过程中的损
	2.  产生阻塞：基于Monitor对象，当对线程同时访问一段同步代码时，首先会进入Entry Set，当一个线程获得对象的锁后，才能进入The Owner区域，其他线程还在继续等待


### 特性

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


关于自适应自旋：
- JDK6后引入自适应自旋机制，通过监控轻量级锁等待情况，动态调整自旋等待时间，如果时间很短说明锁的竞争不激烈，可以线程自旋等待避免线程挂起和恢复带来的性能损失；如果等待时间长，则说明竞争激烈，应该及时释放CPU资源。
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




## 3、AQS

AQS（AbstractQueuedSynchronizer）是为了解决多个线程同时抢占一个或多个资源时出现的并发问题，其同时出现抢占的场景有
1. `ReetrantLock` 对一个资源读写抢占
2. `Semaphore` 多个信号量抢占
3. `CountDownLatch` 并发资源等待通知
4. `ReetrantReadWriteLock` 读写资源

```text
抽象同步队列器是实现锁的关键，在锁的实现中聚合同步器利用同步器实现锁的语义。

可以理解二者之间的关系：
	1. 锁是面向使用者的它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；
	2. 同步器面对的是锁的使用者它简化了锁的实现方式，屏蔽了同步状态管理线程的排队等待与唤醒等底层操作。
锁和同步器很好的隔离了使用者和实现者锁需要关注的领域

```

### 独占锁

**acquire阻塞获取独占锁**

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

1. 首先调用自定义同步器实现的tryAcquire，保证线程安全的校验当前state是否被占用（获取同步状态）
2. 获取失败，则构造同步节点并加入到同步队列器的尾部
3. 最后以死循环的方式获取同步状态，如果获取不到则阻塞节点中的线程而阻塞的只能通过前置节点唤醒或线程中断退出（这也是可支持外部中断实现关键）

**实现同步队列FIFO**

![[image-Concurrent-03 锁结构-20240429020505420.png|500]]


如果现在节点来了发现没有状态同步了就要加入到FIFO队尾挂起等待避免CPU，那么当有机会来了直接通过前置节点通知？

![[image-Concurrent-03 锁结构-20240429020524289.png|500]]




waitStatus = SIGNAL 值为-1，后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或被取消将会通知后续节点，使得后续节点的线程得以运行。

waitStatus = CANCELLED 值为1，由于在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待节点进入该状态将不再变化。

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

会唤醒头节点的后继节点线程，unparkSuccessor方法使用LockSupport来唤醒处于等待状态的线程。

**tryAcquireNanos独占式超时获取同步状态**

提供了传统的synchronized所不具备的特性。

![[image-Concurrent-03 锁结构-20240429020608359.png|600]]



<aside> 💡 **关于interrupt i**nterrupt：实例方法，将线程中断状态标记为true interrupted: 类方法，清楚中断标记并返回当前线程中断状态。即连续两次调用一定返回false isInterrupted: 是否中断

</aside>

### Reentrant重入锁

 - **重入**

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


- **公平**

	nonfairTryAcquire 新加 hasQueuedPredecessors 即加入了同步队列中当前节点是否有前驱节点的判断
	
	非公平可能使得线程发生饥饿的情况，为什么还被设置为默认？极少的线程上下文切换保证了吞吐量

### LockSupport

### Condition

[Untitled Database](https://www.notion.so/31055187e7f749a2aa1cfa96898b6db7?pvs=21)

### 等待队列

等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用。如果调用了Condition.await()方法


![[image-Concurrent-03 锁结构-20240429020637185.png|600]]


在Object的监视器模型上一个对象拥有一个同步队列和等待队列，而并发包的Lock（更确切的说是同步器）拥有一个同步队列和多个等待队列。这也是它与Object在区别公平，超时等之后其能支持多个线程的等待唤醒，能力更强。

<aside> 💡 与Object.wait区别 使用上 **Object.wait** **不能单独使用必须是在synchronized下才能使用；必须通过Notify唤醒** **Condition.await 必须是当前线程被排斥锁lock后才能使用；必须通过sign唤醒**

</aside>