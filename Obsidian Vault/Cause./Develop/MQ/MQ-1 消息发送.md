
## 消息发送


## 消息分布式

### MQ
消息的目的是解耦但彻底的解耦又必然带来一致性的问题

![[MQ-1 消息发送.png|700]]主要步骤如下：
1.  生产者发送 `Half` 不消费的消息（如发送失败，half -> unknow -> 删除）
2.  服务端 Broker 存储完成后，回写给生产者
3.  生产者调用回调方法 `TransactionListener` 执行本地方法
4.  本地执行的结果来决定 Broker 是 `Commit` 还是 `RollBack` 消息
5.  二次确认消息如果 Broker 没收到消息
6.  Producer 回调 Local 确认

### 本地消息表

![[MQ-1 消息发送-2.png]]

在这个过程中，可能几个步骤都发生失败，那失败了怎么办？
 -  1、2失败，因为在同一个事务中，所以会回滚
 -  3 失败，那么就需要有一个定时任务扫描消息数据，对未成功的消息再次投递
 -  4、5失败，则依靠消息重试机制
 -  6、7失败，相当于两个分布式系统的数据已经一致，但本地消息表还是错的。可以定时查询下有状态？？？（如果有很多下游不是累死？）

### 2PC

![[MQ-1 消息发送-1.png|600]]

实现2PC的时候没必要单独启动一个事务协调服务，把这个协调服务的工作最好和订单服务或优惠券放在同一个进程里面这样做有两个好处：
1.  参与分布式事务的进程少故障点就更少，稳定性更好
2.  减少了一些远程调用，性能也更好
其也保证了分布式事务的强一致性适合要求比较高的场景

2PC明显的缺点是 ***整个事务的执行过程需要「阻塞服务端的线程数和数据库」 所以性能不是很高*** 。并且协调者是单节点一旦过程中「协调者宕机」就会导致一直卡在等待提交阶段，这段时间对数据库或服务造成较大影响

### 3PC

### TCC

### Saga
它利用分布式消息来协调本地事务，将各个系统的接受到 **「异步消息」** 后处理结果给协调者通过结果来判断是否继续 向下或回滚之前事务（这里的回滚更多的是更新状态）。
流程很简单但使用它们有一些挑战
1.  Saga之间缺乏隔离
2.  发生错误时回滚更改

### 协同式saga
把Saga的决策和执行顺序逻辑分布在Saga的每一个参与方中，它们通过交换事件的方式来沟通。这样会有一些弊端：
1. 更难理解。没有单一的Saga编排
2. 服务之间的循环依赖
3. 紧耦合风险

### 编排式Saga
Saga的每一个决策和顺序都集中在一个Saga编排器中，通过编排器发出命令式消息给各个Saga参与方。
这里将Saga定义为一个状态机模式更好

### 隔离性问题
在事务的ACID中事务隔离性会带来一些问题，分布式环境下依旧也是如此
1. 丢失更新。一个Saga更新了另一个还未执行完Saga的某一个节点
2. 脏读。读到了另一个Saga没有执行完全的节点数据
3. 不可重复读。一个Saga读到了两个结果
解决方案
1. 语义锁 *_PEDING
2. 版本文件
3. 重读值。乐观锁的一种














## 顺序消息

- **生产者**
RocketMQ 提供了基于队列(分区)的顺序消费，即同一个队列内的消息可以做到有序，但不同队列内的消息是无序的

```java
public static void main(String[] args) throws UnsupportedEncodingException {  
    try {  
        MQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");  
        producer.start();  
  
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};  
        for (int i = 0; i < 100; i++) {  
            int orderId = i % 10;  
            // 同步发送 + 同一个队列
            Message msg =  
                new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,  
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));  
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {  
                @Override  
                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {  
                    Integer id = (Integer) arg;  
                    int index = id % mqs.size();  
                    return mqs.get(index);  
                }  
            }, orderId);  
  
            System.out.printf("%s%n", sendResult);  
        }  
  
        producer.shutdown();  
    } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {  
        e.printStackTrace();  
    }  
}

```


-  **消费者**
[[MQ-3 消息消费#消费顺序]]

