## 1、存储过程

![[MQ-2 消息存储-5.png]]


![[MQ-2 消息存储-6.png]]


### 存储核心
-  CommitLog
	消息存储文件，所有topic消息都存储在CommitLog文件
-  ConsumeQueue
	消息消费队列，消息到达CommitLog文件后，异步转发到消费队列，标记了在CommitLog中的起始offset、log size 和 Message Tag的hashCode ???
-  IndexFile
	消息索引文件，主要存储消息Key与Offset的对应关系

### 存储优缺点

	Kafka 和 RocketMQ 类似，每个Topic有多个 partition(queue)，Kafka 的 partition 是一个独立的物理文件，消息直接从里面读写。根据之前阿里中间件团队的测试，一旦 Kafka 的 Topic partitoin 数量过多，队列文件会过多，会给磁盘的IO读写造成很大的压力，造成tps迅速下降。
	所以RocketMQ进行了上述这样设计，consumerQueue 中只存储很少的数据，消息主体都是通过 CommitLog 来进行读写
	
- **优点**
	1.  队列轻量化，单个队列数据非常少
	2.  对磁盘访问穿行化，避免磁盘竞争
- **缺点**
	1.  写虽然是顺序写，但读却是完全随机读。读消息会先走 ConsumeQueue 再查 CommitLog
- **解决方案**
	1.  随机读尽可能命中 page cache 从而减少IO读操作，所以内存越大越好
	2.  预热 CommitLog 数据

---
## 2、存储架构

![[MQ-2 消息存储-4.png|600]]

### 业务处理
Broker端对消息进行读取和写入逻辑，如前置检查、decode序列化、构造Response等

### 存储组件
核心类 `DefaultMessageStore` 通过方法 `putMessage` 和 `getMessage` 完成对 CommitLog 的日志数据读写操作。另外在组件初始化时还会有 AllocateMappedFileService预分配线程、ReputMessageService 回放存储线程、HAService主从同步线程、IndexService索引文件线程等

### 存储逻辑

### 文件内存封装

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
  内存映射(mmap)是一种内存映射的方法即将一个文件或者其他对象映射到进程的地址空间，实现 ***文件磁盘地址***  和 ***应用程序进程虚拟空间*** 中一段虚拟地址的一一映射关系。实现这种映射关系后进程就可以采用指针的方式读写操作这一段内存。
```

mmap的优势在于，把磁盘文件与进程虚拟地址做了映射，这样可以跳过page cache，只使用一次数据拷贝


### 磁盘存储


---
## 3、延迟消息

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


## 4、消息重试

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


## 5、消息堆积



