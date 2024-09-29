
## 节点注册

-  **初始化连接**
  
	![[MQ-1 消息发送-5.png|500]]
	
	1.  Broker 启动的时候去向所有的 NameServer 注册，并保持长连接，每30s发送一次心跳
	   
	2.  Producer 在发送消息的时候从 NameServer 获取 Broker 服务器地址，根据负载均衡算法选择一台服务器来发送消息
	   
	3.  Conusmer 消费消息的时候同样从 NameServer 获取 Broker 地址，然后主动拉取消息来消费


-  **为什么是 NameServer 而不是 ZK ?**

	1.  复杂度和实现简单性
	   
	2.  一致性与可用性选择：zk选择了CP强一致性而牺牲了一定的可用性，而ns选择了AP可用性，通过心跳机制保证数据的最终一致性
	   
	3.  服务注册和发现：Broker 在启动时会向所有 NameServer 节点进行注册，而不是只向某些节点注册。这样每个 NameServer 节点都会有一份完整的 Broker 注册信息，生产者和消费者可以从任意一个 NameServer 获取完整的信息
	   
	4.  PULL模式：RocketMQ 的 NameServer 采用 PULL 模式，生产者和消费者每隔一段时间主动去 NameServer 拉取信息，而不是 NameServer 推送更新。这种模式简化了 NameServer 的实现，减少了实时性要求


## 消息发送

### 事务消息


消息的目的是解耦但彻底的解耦又必然带来一致性的问题

![[MQ-1 消息发送.png|700]]

主要步骤如下：
1.  生产者发送 `Half` 不消费的消息（如发送失败，half -> unknow -> 删除）
2.  服务端 Broker 存储完成后，回写给生产者
3.  生产者调用回调方法 `TransactionListener` 执行本地方法
4.  本地执行的结果来决定 Broker 是 `Commit` 还是 `RollBack` 消息
5.  二次确认消息如果 Broker 没收到消息
6.  Producer 回调 Local 确认


### 顺序消息

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

