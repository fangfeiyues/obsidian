
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

![[MQ-1 消息发送-1.png|500]]

实现2PC的时候没必要单独启动一个「事务协调者」，给每个参与者发送prepare消息，执行本地数据脚本但不提交事务。

协调者 最好和某个参与者在同一个进程里，这样做有2个好处：
1.   参与分布式事务的进程少故障点就更少，稳定性更好
2.   减少了一些远程调用，性能也更好

2PC看似能提供原子性操作，但存在严重缺陷
-  网络抖动导致的数据不一致。第二阶段commit命令发送后，如果某些参与者接受不到命令则会一直无法提交事务，造成数据的不一致
-  超时导致的同步问题。2PC中参与者节点都是事务阻塞型，当某一个节点出现通信超时，其余参与者都会被动无法释放
-  单点故障风险。 由于严重依赖协调者，一旦发生故障都还处于锁定资源状态

### 3PC

3PC在2PC的第一阶段和第二阶段中插入一个准备阶段，同时协调者引入超时机制
-  CanCommit：协调者向所有参与者发送 CanCommit 命令，是否可以执行
-  PreCommit：协调者向所有参与者发送 PreCommit 命令，锁定资源但不等待避免，直接获取出现卡住情况
-  DoCommit：正式提交事务

### TCC
TCC（ Try-Confirm-Cancel ）补偿事务，其核心思想是针对每个操作都要注册一个与其对应的确认Try和补偿Cancel。与2PC相似
-  Try：下单时通过Try操作扣除库存预留资源
-  Confirm：



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

