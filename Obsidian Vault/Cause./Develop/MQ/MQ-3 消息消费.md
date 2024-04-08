## 背景

```
 最近项目都有遇到在消息量大的情况下，即使是MQ方已经对消息做了分区定量消费控流消费的情况下，因为下游或业务逻辑复杂进行了N*M次的RPC查询或DB插入操作等不免对其他服务造成一定的压力，这个时候我们就需要对MQ的消费速率进行更为灵活的流控
 
 说说其中的一个项目吧，最开始考虑线程池接受消息避免消息一次性的消费情况下耗费大量的服务线程（当然后来发现这是很不可能因为消息本身会因为客户端消费能力的不足而积压），在这种情况下如果消息不断进入线程池的阻塞队列终究也会满的（当然如上满了如果选择主线程执行方式还是会阻塞消费而积压），所以会考虑下使用Tesla的限流方式但是后来想到Tesla这里也是同样阻塞还是会有消息的不断重试问题... 

 如上，其实解决这个问题的本身比较简单直接使用消息自己的消费能力就好了，在客户端消费跟不上的情况下自然会积压的。如果在这个时候我们第三方服务能扛得住完全OK，我们项目是在HTTP查询数据后直接DB其中的数据量大概是 4*9（线程） * 500 （单线程条数） = 3.6w 次插入DB操作那其实控制下还能接受但是如果这个时候是放大N倍的RPC调用那可就有一些危险了... 所以这里看下MQ的消费及限流策略还是很有必要的
```

## 消费模式
### 拉模式Pull

RocketMQ的拉消费模型

![[MQ-3 消息消费.png]]

可以看到消费端的消费模型大致分为三个模块：负载队列，拉取消息，消息消费
1. 服务端 `RebalanceImpl#doRebalance` 根据订阅的每个 topic 循环做消息队列负载
2. 服务端 根据topic在Broker的总队列及订阅tag的客户端，进行负载MessageQueue
3. 消费者在每次负载后会比较&计算本地队列，然后 `MessageQueue + ProcessQueue + nextOffset` 构造成拉取的PullRequest
4. 消费者`PullMessageService`在阻塞队列下等待【3】的数据到来
5. 待补充
6. 消费者利用`PullRequest`开始到 Broker 拉取实时数据，如果中断&队列processQueue缓存消息超过1000&大小超过100M 则会延迟再次消费，同时根据顺序非顺序控制消费...
7. 消费者拉到数据开始消费，否则没消息继续拉取（服务端有长轮询机制）
8. 消费者 `ConsumeMessageConcurrentlyService#submitConsumeRequest` 消费在配置为 「20 无限大 64的线程池中进行... 这样只能保证
9. 本地接口开始接受到消费代码
10. 消费者根据消费结果 ACK 更新 Broker或Local 的消息进度

综上来看对于每个topic的消费者来看，想要达到限流的效果 一方面可以控制内存中的队列多少（大小），一方面在线程池上加以控制避免在全量的消息冲击下服务击垮。
所以可以看到在并发20个线程进行消费时如果后端消费能力不足的情况下会一次阻塞在线程队列中，这时 ProcessQueue 内存待消费消息不断积累阻塞消息的拉取

![[MQ-3 消息消费-1.png|400]]

由上的消息流转流程来看：
1.  `RebalanceImpl` 每20s负载一次topic的 PullRequest 队列
2.  `PullMessageService` 根据负载的队列去 Broker 上拉取数据
3.  `ConsumeMessage` 并发消费数据并更新 ProcessQueue 内存中消息数据

这里对于单个topic的消费来说，每个topic都有4个MessageQueue同时每个MQ每次拉取8条消息，这样 ProcessQueue 每次拉取32条消息的话在线程池中遍历消费。假设我们系统单条消息的消费能力是 40s，这样每分钟就会阻塞66条消息（20s重新负载拉取一次）大约在 15min后内存进度 ProcessQueue 的积累消息就会大于1000条，此时直接返回再等50ms重新拉取



---
## 消费乱序

### 背景

```
    最近的项目渠道版遇到很多消息问题也是其业务本身也是基于消息流转的。如基于销售员订单创建取消消息，销售员创建&解绑消息等，这在分布式环境下就很难保证消息最终消费的一致性而出现不少问题。目前其主要包括两类：
```

1. 消息乱序 + 节点有状态的如订单状态机
2. 消息乱序 + 节点无状态的如创建&删除

### 解决方案

```
   其第一种可以通过在不同的状态之间使用状态模式进行驱动，后一种其实也是殊途同归不过其中消息驱动过程由应用方自己控制当然这只是一种理解方式。另一种方式就是基于中间件自己的能力做到局部的有序性 不过貌似Kafka的单 partition下实现顺序性就是如此... 可以看看RocketMQ的实现方式
```

#### 基于中间件

##### NSQ

```
  NSQ新集群中，消息的顺序生产／消费基于topic partition。Producer（生产者）通过指定shardingID，向目标partition发送消息；Consumer（消费者）通过指定partitionID，从指定partition接收消息。Consumer进行顺序消费时时，Rdy相当于将被NSQ服务端指定为1，在当前消息Finish之前不会推送下一条
```

##### RocketMQ

**生产端**

```java
     // 生产者的消息必须在同一个队列
           for (int i = 0; i < 100; i++) {
                int orderId = i % 10;
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

```

**消费端**

在 ConsumeRequest 进行顺序消费的时候

![[MQ-3 消息消费-2.png]]


顺序消费代码 参考MessageListenerOrderly

```
    这里可以看到在顺序消费下每个消息ProcessQueue对应的一个MessageQueue消息队列。这里每个topic默认对应的4（OR16）个MessageQueue那理解就是topic下的消息在负载之后会有4个队列提供消费，每个队列一个线程。这样的消费能力也太差劲了些... （感觉哪里不对啊这里是所有机器消费4个队列）
```

##### Kafka

总结三种消息队列来说基于中间件的方式都差不多通过控制顺序消息发送到分区partition或队列然后在这个区或队列下单线程(rdy=1)进行消费，如果未finish不会下一条继续推送。从而达到顺序的目的

**不足**
1. 中间件的有序性只能保证在消费的时候有序性而不能保证在消息投递的时候也是一定有序的
2. 消费能力下降单线程基本下降了4倍的消费，尤其是对于订单模块来说很容易造成阻塞
3. 消息异常此分区下的消息都不能消费，风险很高

#### 基于业务

业务分析解决的扩展行和业务把控更好   


---

---


---
## 消费顺序

RocketMQ 的 `MessageListener` 回调函数提供了两种消费模式
-  有序消费模式 `MessageListenerOrderly` 
-  并发消费模式 `MessageListenerConcurrently`
为了保证顺序消费，需要3把锁
1. 消费者对`Broker`加分布式锁，消息只会发给这个Consumer ID
2. 对 `MessageQueue消息队列` 加本地锁，确保同一时间一个队列只有一个线程处理（其他队列）
3. 对 `ProcessQueue消费队列` 加本地锁，保证在broker重平衡后解锁本地ProcessQueue的时候消息处理完了，不会出现到新的Consumer B重复消费

可以看出顺序消费是在消费者上加多次锁实现的，带来的问题就是会降低吞吐量，并且一旦消息阻塞就会导致更多的消息阻塞

![[MQ-3 消息消费-3.png]]


---


## 消费异常



## 延迟消费

DelayQueue