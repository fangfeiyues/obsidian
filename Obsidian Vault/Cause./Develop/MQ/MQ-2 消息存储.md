记一次消息的存储之旅

## 存储过程

### 1、Netty网络传输


### 2、写内存 & 刷磁盘

#### CommitLog

消息存储文件，所有topic消息都存储在CommitLog文件
- **文件结构**
	`CommitLog是RocketMQ消息存储的核心文件，所有topic消息都顺序写入文件中，默认大小为1GB，文件名由20数字组成表示文件的起始偏移量
	`CommitLog的消息结构包括消息头（消息ID、topic、queueId等）、消息体、消息属性（延迟级别、事务相关等）、存储开销

- **内存映射**
	`page cache 页面缓存：是操作系统内核维护的一个内存区域，当RocketMQ写入消息到CommitLog文件时消息数据会先写入到page cache，来提高写入速度减少磁盘IO 以及 保证顺序写入后预读

	`mmap 内存映射：RocketMQ主要通过MappedByteBuffer，直接在内存读写数据而无需读写的系统调用。
	`利用了NIO中的 FileChannel模型 直接将磁盘上的 物理文件 映射到 用户态的内存地址。这种Mmap的方式减少了传统IO将 磁盘文件数据在 操作系统内核地址空间的缓冲区 和 用户应用程序地址空间的缓冲区 之间来回进行拷贝的性能开销，将对文件的操作转化为直接对内存地址进行操作，从而极大地提高了文件的读写效率

#### 刷盘

- **同步刷盘**
	`在这种模式下，消息写入CommitLog后，Broker会等待操作系统将数据刷写到磁盘中的PageCache。一旦数据被刷写到PageCache，Broker会向生产者确认消息已经存储成功。这种方式确保了消息的持久化，即使系统崩溃，消息也不会丢失

- **异步刷盘**
	`在这种模式下，消息写入CommitLog后，Broker会立即向生产者确认消息存储成功，而消息刷写到磁盘的过程是在后台异步进行的。这种方式提高了性能，但在某些极端情况下，如果系统崩溃，可能会丢失尚未刷写到磁盘的消息。

### 3、构建索引文件

#### ConsumeQueue
消息消费队列，消息到达CommitLog文件后，异步转发到消费队列，标记了在CommitLog中的起始offset、log size 和 Message Tag的hashCode ???

#### IndexFile
消息索引文件，主要存储消息Key与Offset的对应关系。通过key或时间区间来查询消息

### 4、HA

![[MQ-2 消息存储-9.png]]


---
## 存储架构

![[MQ-2 消息存储-4.png|600]]

### 1、业务处理
Broker端对消息进行读取和写入逻辑，如前置检查、decode序列化、构造Response等

### 2、存储组件
核心类 `DefaultMessageStore` 通过方法 `putMessage` 和 `getMessage` 完成对 CommitLog 的日志数据读写操作。另外在组件初始化时还会有 AllocateMappedFileService预分配线程、ReputMessageService 回放存储线程、HAService主从同步线程、IndexService索引文件线程等

### 3、存储逻辑

### 4、文件内存封装

MappedByteBuffer 和 FileChannel 完成数据文件读写。

 - DirectByteBuffer
```java
// 开辟对外内存池
public void init() {
        for (int i = 0; i < poolSize; i++) {
			// new DirectByteBuffer(capacity) 也还是直接内存，allocate才是堆内存
            ByteBuffer byteBuffer = ByteBuffer.allocateDirect(fileSize);
            final long address = ((DirectBuffer) byteBuffer).address();
            Pointer pointer = new Pointer(address);
			// 主要是有一层锁定的逻辑
            LibC.INSTANCE.mlock(pointer, new NativeLong(fileSize));
            availableBuffers.offer(byteBuffer);
        }
    }

```

 - MappedByteBuffer
```java
private void init(final String fileName, final int fileSize) throws IOException {
        try {
			// 先生成FileChannel，再对FileChannel做映射
            this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
            this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
        } catch (FileNotFoundException e) {
            log.error("create file channel " + this.fileName + " Failed. ", e);
            throw e;
        } catch (IOException e) {
            log.error("map file " + this.fileName + " Failed. ", e);
            throw e;
        } finally {
            if (!ok && this.fileChannel != null) {
                this.fileChannel.close();
            }
        }
    }
```


![[MQ-2 消息存储.png]]

- 内存映射
```
  内存映射(mmap)是一种内存映射的方法即将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址和应用程序进程虚拟空间中一段虚拟地址的一一映射关系。
  实现这种映射关系后进程就可以采用指针的方式读写操作这一段内存
```

mmap的优势在于，把磁盘文件与进程虚拟地址做了映射，这样可以跳过page cache，只使用一次数据拷贝


-  代码流程
![[MQ-2 消息存储-7.png]]

### 5、磁盘存储

### vs 其他MQ

#### vs Kafka

- **混合型 vs 独用型**

```
「存储结构」 
    RocketMQ采用的是混合型的，即为Broker单个实例下所有的队列共用一个日志数据文件CommitLog来存储。而Kafka采用的是独立型的存储结构，每个队列一个文件

「优缺点」 
	1、RocketMQ优点在于写磁盘串行化处理，避免磁盘竞争；消费队列轻量化，单个数据非常少
	2、RocketMQ缺点在于会存在较多的随机读操作，因此读的效率偏低；
	   同时消费消息需要依赖ConsumeQueue，构建该逻辑消费队列需要一定开销。
    解决方案在于，随机读尽可能命中page cache减少IO操作，还有预热CommitLog数据

```

#### vs NSQ ???



## 存储过程中一些细节处理
### 1、延迟消息

- 思路
	Broker 将消息先存储在内存中，然后使用Timer定时器进行消息延迟，到达指定时间后再存储到磁盘，投递给消费者
- 实现
	1、RocketMQ 在 Broker 端使用一个时间轮来管理定时消息
	2、时间轮的每个槽位对应一个时间间隔，比如1秒、5秒、10秒等，每次时间轮的滴答，槽位向前移动一个时间间隔
	3、当 Broker 接收到定时消息时，根据消息的过期时间计算出需要投递的槽位，并将消息放置到对应
	4、当时间轮的滴答达到消息的过期时间时，时间轮会将槽位中的所有消息投递给消费者
- 时间轮算法
	但正常的Timer定时器是有缺陷的，比如某一时刻有大量任务执行，会导致性能下降影响投递。在RocketMQ 5.0 采用一种基于`时间轮方式`。
	其能在O(1)时间内找到下一个即将执行的任务，并且能支持更高的消息精度。详见： [[时间轮定时器]]

![[MQ-2 消息存储-1.png|600]]


### 2、消息重试

-  重试场景
	RocketMQ规定以下3种情况会发起重试
	1.  业务消费方返回`ConsumeConcurrentlyStatus.RECONSUME_LATER`
	2.  业务消费方返回 null
	3.  业务消费方主动/被动 抛出异常
-  重试流程
	1、Consumer消费的时候，会订阅指定的 `TOPIC-NOMAL_TOPIC` 和 该ConsumerGroup对应的重试`TOPIC-RETRY_GROUP1_TOPIC` ，同时消费来自这两个TOPIC的消息
	2、Consumer消费失败后，调用`sendMessageBack`将失败消息返回Broker
	3、Broker 的 `SendMessageProcessor` 根据当前重试次数确定延时级别，将消息存入延时队列-SCHEDULE_TOPIC中
	4、`ScheduleMessageService` 将到期消息重发到重试队列 `RETRY_GROUP1_TOPIC` 被 Consumer 重新消费 

可以对比之前的延时消息流程，其实重试消息就是将失败的消息当作延时消息进行处理，只不过最后投入的是专门的重试消息队列中

![[MQ-2 消息存储-2.png|600]]


### 3、消息堆积




