## 与Synchronized

同步原理：数据同步需要依赖锁，锁的同步依赖谁呢？Synchronized是在软件层面依赖JVM，juc.Lock是在硬件层面依赖特殊的CPU指令

Synchronized主要三种用法
## AbstractQueuedSynchronizer

AQS是为了解决多个线程同时抢占一个或多个资源时出现的并发问题，其同时出现抢占的场景有

1. ReetrantLock 对一个资源读写抢占；
    
2. Semaphore 多个信号量抢占；
    
3. CountDownLatch 并发资源等待通知；
    
4. ReetrantReadWriteLock 读写资源
    
    ```
     抽象同步队列器是实现锁的关键，在锁的实现中聚合同步器利用同步器实现锁的语义。可以理解二者之间的关系：锁是面向使用者的它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；同步器面对的是锁的使用者它简化了锁的实现方式，屏蔽了同步状态管理线程的排队等待与唤醒等底层操作。锁和同步器很好的隔离了使用者和实现者锁需要关注的领域。
    ```
    

同步器可重写方法

[Overwrite](https://www.notion.so/e4042c8fac68402dad640da02b84b21c?pvs=21)

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

## Reentrant重入锁

### 重入

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

### 公平

在nonfairTryAcquire的地方新加hasQueuedPredecessors即加入了同步队列中当前节点是否有前驱节点的判断

非公平可能使得线程发生饥饿的情况，为什么还被设置为默认？极少的线程上下文切换保证了吞吐量。

## LockSupport

## Condition

[Untitled Database](https://www.notion.so/31055187e7f749a2aa1cfa96898b6db7?pvs=21)

### 等待队列

等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用。如果调用了Condition.await()方法


![[image-Concurrent-03 锁结构-20240429020637185.png|600]]


在Object的监视器模型上一个对象拥有一个同步队列和等待队列，而并发包的Lock（更确切的说是同步器）拥有一个同步队列和多个等待队列。这也是它与Object在区别公平，超时等之后其能支持多个线程的等待唤醒，能力更强。

<aside> 💡 与Object.wait区别 使用上 **Object.wait** **不能单独使用必须是在synchronized下才能使用；必须通过Notify唤醒** **Condition.await 必须是当前线程被排斥锁lock后才能使用；必须通过sign唤醒**

</aside>