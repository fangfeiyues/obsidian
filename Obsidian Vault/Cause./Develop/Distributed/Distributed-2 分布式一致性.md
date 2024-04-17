怎么理解一致性？在分布式系统中，各个节点

## ConsistentHash

### 角色


### 过程


### 落地

Dubbo：负载请求的Provider
RocketMQ：负载消费者的MQ

## Raft
Raft算法是一种易于理解的分布式一致算法，它将一致性问题的复杂性分解为若干个相对独立的子问题。Raft算法通过选举一个领导者（leader）来负责处理所有的写操作，而其他节点作为跟随者（follower）来复制领导者的日志。当领导者出现故障时，跟随者会进行选举产生新的领导者。Raft算法具有强一致性、高可用性和容错性等特点

### 角色


### 过程


### 落地



etcd、redis
 选举的票数大于等于 num(sentinels) / 2 + 1 时，将成为领导者，如果没有超过，继续选举


## Paxos
（帕克索斯）

### 角色

可以理解为 人大代表(Proposer) 在人大向 其它代表(Acceptors) 提案，通过后让 老百姓(Learner) 落实

-  **提议者Proposer**
	提出提案Proposal，Proposal信息包括 提案编号Proposal ID 和 提议的值Value

-  **决策者Acceptor**
	参与决策，回应Proposers的提案。收到Proposal后可以接受提案，若Proposal获得多数Acceptors的接受，则称该Proposal被批准

-  **最终决策学习者Learner**
	不参与决策，从Proposers/Acceptors学习最新达成一致的提案(Value)

### 过程

在多副本状态机中，每个副本同时具有Proposer、Acceptor、Learner三种角色
1. **第一阶段: Prepare阶段**；Proposer向Acceptors发出Prepare请求，Acceptors针对收到的Prepare请求进行Promise承诺。
    1. `Prepare`: Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。
    2. `Promise`: Acceptors收到Prepare请求后，做出“两个承诺，一个应答”。
        1. 承诺1: 不再接受Proposal ID小于等于(注意: 这里是<= )当前请求的Prepare请求;
        2. 承诺2: 不再接受Proposal ID小于(注意: 这里是< )当前请求的Propose请求;
        3. 应答: 不违背以前作出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。

2. **第二阶段: Accept阶段**; Proposer收到多数Acceptors承诺的Promise后，向Acceptors发出Propose请求，Acceptors针对收到的Propose请求进行Accept处理。
    1. `Propose`: Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。
    2. `Accept`: Acceptor收到Propose请求后，在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID和提案Value。

3. **第三阶段: Learn阶段**; Proposer在收到多数Acceptors的Accept之后，标志着本次Accept成功，决议形成，将形成的决议发送给所有Learners。

#### [#](#什么是raft算法)

---

著作权归@pdai所有 原文链接：https://pdai.tech/md/interview/x-interview-2.html

### 落地

kafka、hbase



## Zab

ZooKeeper是一个分布式协调服务 它使用Zab算法来保证数据一致性。
Zab算法在正常运行时采用了类似于Raft算法的领导者-跟随者模式，但在领导者崩溃时，它使用了一种基于消息广播的恢复模式。Zab算法在保证一致性的同时，还提供了高性能和容错性