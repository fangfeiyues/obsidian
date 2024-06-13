
分布式一致性说的都是一个在（主）节点挂了后，怎么不影响性能与可用的情况下继续进行的故事

## 1、ConsistentHash

一致性Hash环的引入是为了解决单调性问题；虚拟节点是为了解决平衡性问题。

	![[image-Distributed-4 分布式一致性-20240418235335548.png|400]]

### 1.1 哈希算法

判定哈希算法好坏的四个定义

- `平衡性(Balance)`: 平衡性是指哈希的结果能够尽可能分布到所有的缓冲中去，这样可以使得所有的缓冲空间都得到利用。很多哈希算法都能够满足这一条件。
    
- `单调性(Monotonicity)`: 单调性是指如果已经有一些内容通过哈希分派到了相应的缓冲中，又有新的缓冲加入到系统中。哈希的结果应能够保证原有已分配的内容可以被映射到原有的或者新的缓冲中去，而不会被映射到旧的缓冲集合中的其他缓冲区。
    
- `负载(Load)`: 负载问题实际上是从另一个角度看待分散性问题。既然不同的终端可能将相同的内容映射到不同的缓冲区中，那么对于一个特定的缓冲区而言，也可能被不同的用户映射为不同的内容。与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低缓冲的负荷。
    
### 1.2 过程


### 1.3 落地

	Dubbo：负载请求的Provider
	RocketMQ：负载消费者的MQ

## 2、Raft

	Raft算法通过选举一个 领导者（leader）来负责处理所有的写操作，而其他节点作为 跟随者（follower）来复制领导者的日志。当 领导者（leader）出现故障时，跟随者（follower）会进行选举产生新的领导者

![[image-Distributed-4 分布式一致性-20240419000223485.png|400]]

### 2.1 角色

Raft将系统中的角色分为 `领导者(Leader)`、`跟从者(Follower)`和`候选人(Candidate)`

-  **领导者 Leader**
	接受客户端请求，并向Follower同步请求日志，当日志同步到大多数节点上后告诉Follower提交日志

-  **跟从者 Follower**
	接受并持久化 Leader 同步的日志，在Leader告之日志可以提交之后，提交日志

-  **候选人 Candidate**
	Leader选举过程中的临时角色

### 2.2 过程

Raft 使用 心跳(heartbeat) 触发Leader选举。当服务器启动时，初始化为 Follower。Leader 向所有 Followers周期性发送 heartbeat。如果 Follower 在选举超时时间内没有收到 Leader 的 heartbeat，就会等待一段随机的时间后发起一次Leader选举。Follower 将其当前term 加一然后转换为 Candidate。它首先给自己投票并且给集群中的其他服务器发送结果有以下三种情况:
-  赢得了多数的选票，成功选举为Leader
-  收到了Leader的消息，表示有其它服务器已经抢先当选了Leader
-  没有服务器赢得多数的选票，Leader选举失败，等待选举时间超时后发起下一次选举

### 2.3 落地

1.  Redis [[Redis-3 分布式#Sentinel 哨兵]]
2.  Etcd


## 3、Paxos
（帕克索斯）

![[image-Distributed-4 分布式一致性-20240419000311619.png]]

### 3.1 角色

可以理解为 人大代表(Proposer) 在人大向 其它代表(Acceptors) 提案，通过后让 老百姓(Learner) 落实

-  **提议者 Proposer**
	提出提案Proposal，Proposal信息包括 提案编号Proposal ID 和 提议的值Value

-  **决策者 Acceptor**
	参与决策，回应Proposers的提案。收到Proposal后可以接受提案，若Proposal获得多数Acceptors的接受，则称该Proposal被批准

-  **最终决策学习者 Learner**
	不参与决策，从Proposers/Acceptors学习最新达成一致的提案(Value)

### 3.2 过程

多副本状态机中，每个副本同时具有 Proposer、Acceptor、Learner 三种角色
 
```
基本的 Paxos 算法非常简单，它由三个角色组成。
1. Proposer：Proposer 可以有多个，Proposer 提出议案（value）。所谓 value，可以是任何操作，比如“设置某个变量的值为 value”。不同的 Proposer 可以提出不同的 value。但对同一轮 Paxos 过程，最多只有一个 value 被批准。
2. Acceptor：Acceptor 有 N 个，Proposer 提出的 value 必须获得 Quorum 的 Acceptor 批准后才能通过。Acceptor 之间完全对等独立。
3. Learner：上面提到只要 Quorum 的 Accpetor 通过即可获得通过，那么 Learner 角色的目的就是把通过的确定性取值同步给其他未确定的 Acceptor

```


### 3.3 Multi-Paxos

这三个角色其实已经描述了一个值被提交的整个过程。其实基本的 Paxos 只是理论模型，因为在真实场景下，我们需要处理许多连续的值，并且这些值都是并发的。如果完全执行上面描述的过程，那性能消耗是任何生产系统都无法承受的，因此我们一般使用的是 Multi-Paxos。

Multi-Paxos 可以并发执行多个 Paxos 协议，它优化的重点是把 Propose 阶段进行了合并，这就引入了一个 Leader 的角色，也就是领导节点。而后读写全部由 Leader 处理，同时这里与 ZAB 类似，Leader 也有任期的概念，Leader 与其他节点之间也用心跳进行互相探活。是不是感觉有那个味道了？后面我就会比较两者的异同。

另外 Multi-Paxos 引入了两个重要的概念：`replicated log` 和 `state snapshot`
1. `replicated log`：值被提交后写入到日志中。这种日志结构除了提供持久化存储外，更重要的是保证了消息保存的顺序性。而 Paxos 算法的目标是保证每个节点该日志内容的强一致性。
2. `state snapshot`：由于日志结构保存了所有值，随着时间推移，日志会越来越大。故算法实现了一种状态快照，可以保存最新的日志消息。当快照生成后，我们就可以安全删除快照之前的日志了。


### 34 落地

 - **Kafka ?**
 
```
Kafka通过分区副本来提高数据的可靠性。
每个主题（Topic）可以分成多个分区（Partition），每个分区可以有一个或多个副本（Replica），其中一个副本是Leader，其他的是Follower。所有的读写操作都通过Leader进行，Follower会定期从Leader同步数据。如果Leader失败，会从Follower中选举出一个新的Leader
```


-  **Hbase**

	Google的Chubby、Spanner，以及IBM的SVC
## 4、Zab

ZooKeeper是一个分布式协调服务 它使用Zab算法来保证数据一致性。

Zab算法在正常运行时采用了类似于Raft算法的领导者-跟随者模式，但在领导者崩溃时，它使用了一种基于消息广播的恢复模式。Zab算法在保证一致性的同时，还提供了高性能和容错性