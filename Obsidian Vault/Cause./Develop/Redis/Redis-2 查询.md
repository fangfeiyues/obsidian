
## 数据结构

### 简单动态字符串 SDS

动态字符串不仅可以用于存储简单字符串，还可以用于实现复杂的列表list、哈希hash、集合set等，以及作为缓冲区buffer如AOF缓冲区以及客户端状态中的缓冲区。
##### 字符串长度
C字符串不记录自身的长度，所以获取必须遍历。但SDS有len属性，可以常数级
##### 缓冲区溢出
```shell
-- 在 s 后拼接字符串
sdscat(s, "Cluster")
```

在执行拼接前会检查 s 的长度是否足够，不足则会扩展 s 的空间
##### 堕性空间释放

当 SDS 的 API 需要缩短 SDS 保存的字符串时，程序并不立即使用内存重分配来回收缩短后的字节，而是使用 free 属性将这些字节的数量记录起来，并等待将来使用



### 链表 Linked

#### 使用
基础类型 List 列表用的 链表 结构

```shell
> lpush fruits apple banana 
> rpush fruits orange
> lpop fruits 
> rpop fruits
```

#### 构造
```c
typedef struct listNode {
   // 前置节点
   struct listNode * prev;
   // 后置节点
   struct listNode * next;
   // 节点的值
   void * value;
}
```

Redis的链表实现的特性可以总结如下：
-  双端：链表节点带有 prev 和 next 指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)
-  无环：表头节点的 prev指针 和 表尾节点的 next指针 都指向NULL
-  带链表长度计数器：获取链表中节点数量的复杂度为O(1)
-  多态：可以用于保存各种不同类型的值

### 字典 Dict

#### 使用
Redis的 String字符串、Hash哈希 以及 一些高级的数据类型如Steams、Bitmap、HyperLogLog等

![[Redis-2 字典.png|500]]
#### 构造
字典使用哈希表作为底层实现，一个哈希表里可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

```c
// --- 字典 ---
typedef struct dict{
  // 类型特定函数
  dictType *type;
  // 私有数据
  void *privdata;
  // 哈希表，重要！！
  dictht ht[2];
  // rehash索引
  int trehashidx;
}

//  --- 哈希表 ---
typedef struct dictht {
   // 哈希表数组，指向 dictEntry 结构的键值对
   dictEntry **table;
   // 哈希表大小
   unsigned long size;
   // 哈希表大小掩码，用于极端索引值
   unsigned long sizemark;
   // 该哈希表已有节点的数量
   unsigned long used;
} dictht;

typedef struct dictEntry {
   // 键
   void *key;
   // 值：可以是一个指针、或者一个uint64_t整数等
   union{ ... }v;
   // 指向下个哈希表
   struct dictEntity *next;
} dictEntity;


```


![[Redis-2 字典.png]]
> 这里有4层指针，其实可以直接从 字典dict 链接到 键值dictEntity指针 即可，但为了渐变和统计还是加了层 dictht 

#### 哈希
当要将一个新的键值对添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后再根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。

##### 哈希算法

Redis 使用 MurmurHash2 算法来计算键的哈希值

##### 哈希冲突
被分配到同一个索引伤的多个节点可以用这个单向链表连接起来，解决键冲突问题

##### rehash重新散列
扩张 或 收缩到一个新的哈希表 `ht[1] = ht[0].used` 的 2的n次方，并释放老的哈希表

##### 渐进式 rehash
在如果直接操作 ht[0] --> ht[1] ，会导致服务在一段时间内的不可用，为了减少影响通过多次、渐进的慢慢rehash到ht[1]，具体操作如下
1.  为 ht[1] 分配空间，让字典同时持有 ht[0] 和 ht[1] 两个哈希表
2.  在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为0，表示 rehash 工作正式开始
3.  在 rehash 期间，每次对字典执行添加、删除、查找或者更新操作时，程序出了执行指定操作外，还会顺带将 ht[0] 哈希表在 rehashidx 索引伤的所有键值对 rehash 到 ht[1]，当工作完成后程序将rehashidx 增1 
4.  所有键值都被 rehash 到 ht[1] 后程序将 rehashidx 设置为-1，表示操作完成


### 跳跃表 ZSkipList
跳表的优势是能支持平均 O(logN) 复杂度的节点查找，支持高效的范围查询
跳表首先是链表，但与传统链表相比有几点差异：
1.  元素按照升序排列存储，跳表结构中会包含排序所需的值
2.  前向节点可能包含多个指针，指针跨度不同

#### 使用
有序集合，支持以下能力
1.  查询有序范围

```shell
redis> ZADD fruits-price 5 'banana' 6.5 'cherry'
> ZRANGE fruits-price 0 2 WITHSCORES
> banana
> 5
> cherry
> 6.5

```

#### 构造
Redis 的跳跃表由 `zskiplistNode` 和 `zskiplist` 两个结构定义，其中 `zskiplistNode` 结构用于表示 **跳跃表节点**，而 `zskiplist` 用于保存跳跃表节点的相关信息如节点的数量以及指向表头节点和表尾节点指针等

![[Redis-2 查询.png|600]]

表节点信息 `zskiplist` 有
1.  header：指向跳跃表的表头节点
2.  tail：指向跳跃表的表尾节点
3.  level：记录跳跃表内最大节点的层数
4.  length：记录跳跃表的长度

跳跃表`zskiplistNode` 有
1.  level：层，每层有两个属性 
	1.  前进指针
	2.  跨度：跨度越大距离越远
2.  backward：后退指针
3.  score：分值，跳跃表所有节点都按分值从小到大排序
4.  obj：成员对象 ，是个指针，指向一个字符串对象

![[Redis-2 查询-1.png]]

##### 层数的设置
跳表的相邻两层的节点数量最理想比例是 2:1 ，查找复杂度可以降低到 O(logN)
Redis的做法是，跳表在创建节点时候，会生成范围为[0-1]的一个随机数，如果这个随机数小于0.25（相当于概率25%），那么层数就增加1层，然后继续生成下一个随机数

#### 为什么不是红黑树

1.  内存占用来说，跳表更灵活。平衡树每个节点2个指针，跳表每个节点为25%即1.25
2.  范围查找上，跳表更有优势
3.  算法实现难度，跳表更简单


### 整数集合 InSet

当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现
```shell
redis> SADD numbers 1 3 5 7 9

```

整数集合的底层实现是数组，这个数组以有序、无重复的方式保存集合元素

### 压缩列表

压缩列表（ziplist）是列表键和哈希键的底层实现之一。当列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串



### 对象



![[Redis-2 查询-2.png|600]]





## Q&A

