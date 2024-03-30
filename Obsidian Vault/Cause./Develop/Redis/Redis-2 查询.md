
## 数据结构

### 链表

#### 使用

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

### 字典

#### 使用
Redis的 字符串String、哈希Hash 以及 一些高级的数据类型如Steams、Bitmap、HyperLogLog等

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


### 跳跃表

#### 使用
有序集合

```shell
redis> ZADD fruits-price 5 'banana' 6.5 'cherry'
> ZRANGE fruits-price 0 2 WITHSCORES
> banana
> 5
> cherry
> 6.5

```

#### 构造

