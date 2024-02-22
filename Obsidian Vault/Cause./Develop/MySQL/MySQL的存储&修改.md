MySQL存储核心要解决的几个问题（ 持续更新ing... ）
1. 怎么支持ACID？也就是写入过程保证原子性、持久性、隔离性和一致性 ...
2.  怎么支持高并发流量？写速度怎么保证、有没有缓存、怎么解决读写一致问题 ... 
3.  怎么保证高可用？宕机后怎么恢复、水平扩展怎么做的 ... 


2024-02-21
# 数据的存储

在 [[MySQL 客户端]] 提到存储引擎即是提供对数据存储服务。先聊聊MySQL常用的InnoDB表记录器。（说是引擎纯粹是为了更高大上～）

## InnoDB页
InnoDB存储引擎需要一条一条的把记录从磁盘上读出来么？不，那样会慢死。InnoDB采取的方式是：将数据划分为若干个页，以 页 作为 磁盘 和 内存 之间交互的基本单位，页的大小一般为 _**16**_ KB。

### 数据页结构
1.  File Header：38字节，？
2.  Page Header：56字节
3.  Infimum + supermum：
4.  User Records：大小不确定，存储记录的核心字段。大小从Free区域部分迁移过来
5.  Free Records：大小不确定
6.  Page Space：大小不确定
7.  File Trailer：8字节




## InnoDB行
InnoDB支持了4种不同类型的行格式，分别是 `Compact`、`Redundant`、`Dynamic` 和 `Compressed` 
```mysql
mysql> CREATE TABLE record_format_demo (
    ->     c1 VARCHAR(10),
    ->     c2 VARCHAR(10) NOT NULL,
    ->     c3 CHAR(10),
    ->     c4 VARCHAR(10)
    -> ) CHARSET=ascii ROW_FORMAT=COMPACT;
```


### COMPACT行格式

![[Compact行格式.png|500]]

#### 额外信息
###### 变长字段长度
MySQL支持的一些变长的数据类型如VARCHAR(M)、VARBINARY(M)、各种TEXT类型等。

###### NULL值列表

###### 记录头信息
- 2个预留位
- `delete_mask`：该记录是否被删除
- `min_rec_mask`：B+树的每层非叶子节点种最小记录都会添加该标记
- `n_owned`：当前记录拥有的记录数
- `heap_no`：当前记录在记录对的位置信息
- `record_type`：记录类型（0 - 普通记录；1 - B+树非叶子节点记录；2 - 最小记录；3 - 最大记录）
- `next_record`：下一条记录的相对位置

#### 真实数据
MySQL会为每个记录默认添加一些隐藏列，如
- `row_id`：唯一标识一条记录
- `transaction_id`：事务ID
- `roll_pointer`：回滚指针

### Redundant行格式
是MySQL5.0前的格式，比较老了。
