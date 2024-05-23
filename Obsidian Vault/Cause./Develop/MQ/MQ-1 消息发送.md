
## 节点注册

![[MQ-1 消息发送-5.png|500]]

1.  Broker启动的时候去向所有的NameServer注册，并保持长连接，每30s发送一次心跳
2.  Producer在发送消息的时候从NameServer获取Broker服务器地址，根据负载均衡算法选择一台服务器来发送消息
3.  Conusmer消费消息的时候同样从NameServer获取Broker地址，然后主动拉取消息来消费

-  **为什么用的NameServer而不是ZK？**

## 消息发送


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

