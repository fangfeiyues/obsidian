## 存储过程


---
## 存储架构

### Buffer

RocketMQ里使用的都是堆外内存池
#### DirectByteBuffer

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


#### MappedByteBuffer

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

### 内存映射

```
  内存映射(mmap)是一种内存映射的方法即将一个文件或者其他对象映射到进程的地址空间，实现 ***文件磁盘地址***  和 ***应用程序进程虚拟空间*** 中一段虚拟地址的一一映射关系。实现这种映射关系后进程就可以采用指针的方式读写操作这一段内存。
```

mmap的优势在于，把磁盘文件与进程虚拟地址做了映射，这样可以跳过page cache，只使用一次数据拷贝




---


---
## 延迟消息

- 思路
	Broker 将消息先存储在内存中，然后使用Timer定时器进行消息延迟，到达指定时间后再存储到磁盘，投递给消费者
- 实现
	1、RocketMQ 在 Broker 端使用一个时间轮来管理定时消息
	2、时间轮的每个槽位对应一个时间间隔，比如1秒、5秒、10秒等，每次时间轮的滴答，槽位向前移动一个时间间隔
	3、当 Broker 接收到定时消息时，根据消息的过期时间计算出需要投递的槽位，并将消息放置到对应
	4、当时间轮的滴答达到消息的过期时间时，时间轮会将槽位中的所有消息投递给消费者

但正常的Timer定时器是有缺陷的，比如某一时刻有大量任务执行，会导致性能下降影响投递。在RocketMQ 5.0 采用一种基于`时间轮方式`。
其能在O(1)时间内找到下一个即将执行的任务，并且能支持更高的消息精度。详见： [[时间轮定时器]]

![[MQ-2 消息存储-1.png|600]]