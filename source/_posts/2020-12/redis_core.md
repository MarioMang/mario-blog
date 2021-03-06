---
title: Redis Core
date: '2020-12-26 20:00:00'
category: Redis
author: MarioMang
cover: /resources/images/default_cover.gif
updated: '2021-01-06 21:00:00'
---

## Redis Core

### 字符串

  Redis构建了一种简单动态字符串(Simple Dynamic String, SDS), 当Redis需要一个可以被修改的字符串值时, Redis就会使用SDS来表示字符串值
  SDS还被用作缓冲区: AOF模块中的AOF缓冲区, 以及客户端状态中的输入缓冲区

  SDS的定义:

  ``` C
  struct sdshdr{
     int len;    // 记录buf数组中已使用字节的数量
     int free;   // 记录buf数组中未使用字节的数量
     char buf[]; // 字节数组, 用于保存字符串
  }

  typedef char *sds;
  ```

  SDS遵循C字符串以空字符('\0')结尾, 空字符不计算在len中

  SDS与C字符串的区别

  1. O(1)获取字符串长度
    C字符串获取字符串长度 size_t strlen(const char *s) 计算, 程序需要遍历整个字符串, 直到遇到'\0'才能计算字符串长度
    SDS字符串只要访问SDS的len属性即可获取到字符串长度
  2. 杜绝缓冲区溢出
    C字符串不记录自身长度, 所以很容易造成缓冲区溢出
    SDS字符串的空间分配策略, 完全杜绝了缓冲区溢出的可能性
  3. 空间预分配
    对SDS进行修改时, 如果SDS的长度小于1MB, SDS会分配和len属性同样大小的未使用空间
    如果SDS的长度大于1MB, SDS会分配1MB的未使用空间
  4. 惰性空间释放
    当SDS需要缩短SDS保存的字符串时, 并不会立即使用内存分配回收缩短后多出来的字节, 而是使用free字段将这些字节数量记录起来

### 链表

  Redis构建了自己的链表

  ListNode定义:

  ``` C
  typedef struct listNode {
    struct listNode *prev; // 前置节点
    struct listNode *next; // 后置节点
    void *value;           // 节点值
  } listNode;
  ```

  List定义:
  
  ``` C
  typedef struct list {
      listNode *head;                       // 表头节点
      listNode *tail;                       // 表尾节点
      void *(*dup)(void *ptr);              // 节点值复制函数
      void (*free)(void *ptr);              // 节点值释放函数
      int (*match)(void *ptr, void *key);   // 节点值对比函数
      unsigned long len;                    // 链表所包含的节点数量
  } list;
  ```

  Redis链表实现的特性:

  * 双端:
    链表节点带有 prev 和 next 指针, 获取某个节点的前置节点和后置节点的复杂度都是O(1)
  * 无环:
    表头节点的 prev 指针和表尾节点 next 指针都指向 NULL, 对链表访问以 NULL 为终点
  * 带表头指针和表尾指针: 
    通过list结构的head指针和tail指针, 程序获取链表的表头节点和表尾节点的复杂度为O(1)
  * 带并链表长度计数器: 
    程序使用list结构的len属性来对list持有的链表节点进行计数, 程序获取链表中节点数量的复杂度O(1)
  * 多态: 
    链表节点使用 void* 指针来保存节点值, 并且通过list结构的dup\free\match三个属性为节点值设置类型特定函数, 所以链表可以用于保存不同类型值

### 字典

  字典, 又称为符号表(symbol table), 关联数组(associative array)或映射(map), 是一种用于保存键值对(key-value pair)的抽象数据结构

  Redis字典使用哈希表作为底层实现, 一个哈希表里面可以由多个哈希表节点, 而每个哈希表节点就保存了字典中的一个键值对

  哈希表定义:

  ``` C
  typedef struct dictht {
    dictEntry **table;      // 哈希表数组
    unsigned long size;     // 哈希表大小
    unsigned long sizemask; // 哈希表大小掩码, 用来计算索引值, 总是等于 size-1
    unsigned long used;     // 该哈希表已有节点的数量
  } dictht;
  ```

  哈希表节点:

  ``` C
  typedef struct dictEntry {
    void *key;   // 键

    // 值
    union {
      void *val;
      uint64_t u64;
      int64_t s64;
    } v;

    struct dictEntry *next; // 指向下个哈希表节点, 形成链表
  } dictEntry;
  ```

  字典定义:

  ``` C
  typedef struct dict {

    dictType *type; // 类型特定函数

    void *privdata; // 私有数据

    dictht ht(2);   // 哈希表

    int trehashidx; // rehash索引
  } dict;
  ```

  ``` C
  typedef struct dictType {

    unsigned int (*hashFunction) (const void *key);      // 计算哈希值的函数

    void *(*keyDup) (void *privdata, const void *key);   // 复制键的函数

    void *(*valDup) (void *privdata, const void *obj);   // 复制值的函数

    int (*keyCompare) (void *privdata, const void *key1, const void *key2); //对比键的函数

    void (*keyDestructor) (void *privdata, void *key);    // 销毁键的函数

    void (*valDestructor) (void *privdata, void *obj);    // 销毁值的函数
  } dictType;
  ```

  哈希算法
    当要将一个新的键值对添加到字典里面时, 程序需要先根据键值对的键计算出哈希值和索引值, 然后再根据索引值, 将包含新键值对的哈希表节点放到哈希表数组指定的索引上面 

  解决键冲突
    Redis的哈希表使用链地址法(separate chaining)来解决键冲突, 每个哈希表节点都有一个next指针, 多个哈希表节点可以用next指针构成一个单向链表, 被分配到同一个索引上的多个节点

  rehash
    Redis对字典的哈希表执行rehash的步骤如下

  1. 为字典的ht[1]哈希表分配空间, 这个哈希表的空间大小取决于要执行的操作, 以及ht[0]当前包含的键值对数量
  2. 将所有保存在ht[0]中的所有键值对rehash到ht[1]上面
  3. 当ht[0]包含的所有键值对都迁移到了ht[1]之后, 释放ht[0], 将ht[1]设置为ht[0], 并在ht[1]创建一个空白哈希表
  
### 跳跃表

  Redis 使用跳跃表作为有序集合键的底层实现之一, 如果有一个有序集合包含的元素数量比较多, 又或这有序集合中元素的成员是比较长的字符串时, Reids就会使用跳跃表来作为有序集合键的底层实现

  跳跃表实现

  ``` C
  typedef struct zskiplistNode {

    struct zskiplistNode *backward; // 后退指针

    double score;                   // 分值

    robj *obj;                      // 成员对象

    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进指针

        unsigned int span;             // 跨度
    } level[]; // 层
  } zskiplistNode;
  ```

  1. 层
    跳跃表节点的level数组可以包含多个元素, 每个元素都包含一个指向其他节点的指针

  2. 前进指针
    每个层都有一个指向表尾方向的前进指针, 用于从表头指向表尾方向访问节点

  3. 跨度
    层的跨度用于记录两个节点之间的距离
  
  4. 后退指针
    节点的后退指针用于从表尾向表头方向访问节点

  5. 分值和成员
    节点的分值和一个double类型的浮点数, 跳跃表中的所有节点都按照分支从小到大来排序
    节点的成员对象是一个指针, 它指向一个字符串对象, 而字符串对象则保存一个SDS值

  跳跃表
    
  ``` C
  typedef struct zskiplist {
    struct skiplistNode *header, *tail; // 表头节点, 表尾节点

    unsigned long length;               // 表中节点数量

    int level;                          // 表中层数最大的节点层数
  }
  ```

### 整数集合

  整数集合是集合键的底层实现, 当一个集合只包含整数值元素, 并且这个集合的元素数量不多时, Redis就会使用整数集合作为集合键的底层实现

  整数集合

  ``` C
  typedef struct intset {
      
      uint32_t encoding;  // 编码方式

      uint32_t length;    // 集合包含的元素数量

      int8_t contents[];  // 保存元素的数组
  } intset;
  ```

  * 如果encoding属性的值为INTSET_ENC_INT16, 那么contents就是一个int16_t[];
  * 如果encoding属性的值为INTSET_ENC_INT32, 那么contents就是一个int32_t[];
  * 如果encoding属性的值为INTSET_ENC_INT64, 那么contents就是一个int64_t[];

### 压缩列表
  
  压缩列表(ziplist)是列表键和哈希键的底层实现, 当一个列表键只包含少量列表项, 并且每个列表项要么就是小整数值, 要么就是长度比较短的字符串

  压缩列表构成:

  |属性|类型|长度| 用途 |
  |:---:|:---:|:---:|:---:|
  |zlbytes|uint32_t|4byte|记录整个压缩列表占用的内存字节数|
  |zltail|uint32_t|4byte|记录压缩列表表尾节点距离压缩列表的起始地址有多少字节|
  |zllen|uint16_t|2byte|记录了压缩列表包含的节点数量|
  |entryX|列表节点|不定|压缩列表包含的各个节点|
  |zlend|uint8_t|1byte|特殊值0xFF, 用于标记压缩列表的末端|

  压缩列表的节点的构成

  字节数组可以是以下3中长度:
  * 长度小于等于63字节的字节数组
  * 长度小于等于16383字节的字节数组
  * 长度小于等于4294967295字节的字节数组

  整数值可以是以下六种长度之一
  * 4位长, 介于0-12的无符号整数
  * 1字节长的有符号整数
  * 3字节长的有符号整数
  * int16_t类型整数
  * int32_t类型整数
  * int64_t类型整数

### 对象

  Redis使用对象来表示数据库中的键和值, 当Redis中创建一个键值对时, 至少会创建两个对象, 一个对象用作键值对的键, 另一个用作键值对的值对象

  | 类型常量 | 名称 |
  |:---:|:---:|
  |REDIS_STRING| 字符串对象 |
  |REDIS_LIST  | 列表对象   |
  |REDIS_HASH  | 哈希对象   |
  |REDIS_SET   | 集合对象   |
  |REDIS_ZSET  | 有序集合   |

  对象编码

  | 编码常量 | 编码对应的底层数据结构 |
  |:---:|:---:|
  |REDIS_ENCODING_INT           |  long类型整数 |
  |REDIS_ENCODING_EMBSTR        |  embstr编码的简单动态字符串 |
  |REDIS_ENCODING_RAW           |  简单动态字符串 |
  |REDIS_ENCODING_HT            |  字典 |
  |REDIS_ENCODING_LINKEDLIST    |  双向链表 |
  |REDIS_ENCODING_QUICKLIST     |  双向链表 |
  |REDIS_ENCODING_ZIPLIST       |  压缩列表 |
  |REDIS_ENCODING_INTSET        |  整数集合 |
  |REDIS_ENCODING_SKIPLIST      |  跳跃表和字典 |

  类型对象

  | 类型 |  编码 | 对象  |
  |:---:|:---:|:---:|
  |REDIS_STRING|REDIS_ENCODING_INT|使用整数值实现的字符串对象|
  |REDIS_STRING|REDIS_ENCODING_EMBSTR|使用embstr编码的SDS|
  |REDIS_STRING|REDIS_ENCODING_RAW | SDS实现字符串|
  |REDIS_LIST  |REDIS_ENCODING_QUICKLIST | 快速列表实现列表对象 |
  |REDIS_HASH  |REDIS_ENCODING_ZIPLIST | 压缩列表实现哈希对象 |
  |REDIS_HASH  |REDIS_ENCODING_HT   | 字典实现哈希对象 |
  |REDIS_SET   |REDIS_ENCODING_INTSET  | 整数集合实现集合对象 |
  |REDIS_SET   |REDIS_ENCODING_HT  | 字典实现集合对象 |
  |REDIS_ZSET  |REDIS_ENCODING_ZIPLIST | 压缩列表实现有序集合对象 |
  |REDIS_ZSET  |REDIS_ENCODING_SKIPLIST | 跳跃表和字典实现有序集合|

  | Redis对象格式 |
  |:---:|
  |redisObject |
  |type REDIS_STRING |
  |encoding REDIS_ENCODING_INT |
  |ptr |
  |...|

#### 字符串对象

  字符串对象的编码可以是int, raw, embstr

* 如果字符串对象保存的是整数值, 并且这个整数值可以用long类型表示, 那么字符串对象会将整数值保存在字符串对象结构的ptr属性里面, 并将字符串对象的编码设置为int
* 如果字符串保存的是字符串值, 并且这个字符串的长度大于44字节, 那么字符串对象将使用一个简单动态字符串来保存字符串值, 并将编码设置为raw格式
* 如果字符串保存的是字符串值, 并且字符串长度小于等于44字节, 那么字符串将编码设置为embstr
  > sds需要分配两次内存, 分别为redisObject和sdshdr 结构来表示字符串对象,
  > embstr则只需要分配一次内存, 空间中一次包含redisObject和sdshdr

#### 列表对象

##### *以下内容在3.2版本之前有效*

  列表对象的编码可以是ziplist或linkedlist

* ziplist编码的列表对象使用压缩列表作为底层实现, 每个压缩列表节点保存了一个列表元素
* linkedlist编码的列表对象使用双向链表作为底层实现, 每个双向链表节点都保存了一个字符串对象, 而每个字符串对象都保存了一个列表元素

当列表对象可以同时满足以下两个条件时, 列表对象使用ziplist编码:

* 列表对象保存的所有字符串元素的长度都小于64字节
* 列表对象保存的元素数量小于512个
不满足以上两个条件的列表对象使用linkedlist编码

##### *Redis3.2版本之后*

  使用quicklist代替原来的方式, quicklist: A doubly linked list of ziplists
  Quicklist是由ziplist组成的双向链表, 链表中的每个节点都以压缩列表ziplist的结构保存

#### 哈希对象

  哈希对象的编码可以是ziplist或者hashtable

* ziplist编码的哈希对象使用压缩列表作为底层实现
  * 保存键值对的两个节点总是在一起, 保存键的节点在前, 保存值的节点在后
  * 先添加的哈希对象会被放在压缩列表的表头方向, 后添加的会被放在表尾
* hashtable编码的哈希对象使用字典作为底层实现
  * 字典的每个键都是一个字符串对象
  * 字典的每个值都是一个字符串对象

当哈希对象同时满足以下两个条件时, 使用ziplist编码:

* 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节
* 哈希对象保存的键值对数量小于512个

#### 集合对象

  集合对象的编码可以是intset和hashtable

当集合对象同时满足以下两个条件, 对象使用intset编码:

* 集合对象保存的所有元素都是整数
* 集合对象保存的元素数量不超过512个

#### 有序集合对象

  有序集合的编码可以是ziplist或者skiplist

当有序集合同时满足以下两个条件时, 对象使用ziplist:

* 有序集合保存的所有元素成员的长度都小于64字节
* 有序集合保存的元素数量小于128个

### 数据库

  Redis服务器将所有数据库都保存在redis.h/redisServer结构的数组中

#### 切换数据库
  
  每个Redis客户端都有自己的目标数据库, 每当客户端执行数据库写命令或者数据库读命令的时候, 目标数据库就会称为这些命令的操作对象
  默认情况下, Redis客户端的目标数据库为0号数据库, 可以通过select命令切换目标数据库

#### 数据键空间(key space)

  Redis是一个键值对数据库服务器, 服务器中的每个数据库都由一个redis/redisDb结构表示, 其中dict字段保存了数据库中的所有键值对, 我们将这个字典称为键空间

* 键空间的键也就是数据库的键, 每个键都是一个字符串对象
* 键空间的值也就是数据库的值, 每个值可以是字符串对象, 列表对象, 哈希对象, 集合对象或有序集合对象

键空间操作

* 添加新键
* 删除键
* 更新键
* 对键取值

读写键空间时的维护操作

* 在读取一个键之后, 服务器会根据键是否存在来更新服务器中键空间命中次数或键空间不命中次数
  > 通过INFO stats 中, keyspace_hits和keysapce_misses属性查看
* 在读取一个键之后, 服务器会更新键的LRU(最后一次使用)时间, 这个值可以用于计算键的空闲时间
  > 通过OBJECT idletime key可以查看键的空闲时间
* 如果服务器在读取一个键时发现该键已经过期, 那么服务器会先删除这个过期键, 然后才执行余下的其他操作
* 如果有客户端使用WATCH命令监视了某个键, 那么服务器在对被监视的键进行修改之后, 会将这个键标记为dirty，从而让事务程序注意到这个键已经被修改
* 服务器每次修改一个键之后, 都会对dirty键计数器的值增1, 这个计数器会触发服务器的持久化以及复制操作
* 如果服务器开启了数据库通知功能, 那么在对键进行修改之后, 服务器将按配置发送相应的事件通知

设置键的生存时间或过期时间

* 设置过期时间
  EXPIRE, PEXPIRE, EXPIREAT, PEXPIREAT都是使用PEXPIREAT命令来实现
* 保存过期时间
  redisDb结构的expires字典保存了数据库中所有键的过期时间, 被称为过期字典
  * 过期字典的键是一个指针, 这个指针指向键空间中的某个对象
  * 过期字典的值是一个long long类型的整数, 这个整数保存了键锁指向的数据库键的过期时间--毫秒精度的UNIX时间戳
* 移除过期时间
  PERSIST命令可以移除一个键的过期时间
* 计算并返回剩余时间
  TTL命令以秒为单位返回键的剩余生存时间, PTTL则以毫秒为单位返回键的剩余生存时间
* 过期键的判定
  1. 检查给定键是否存在于过期字典中, 如果存在则取得键的过期时间
  2. 检查当前UNIX时间戳是否大于键的过期时间, 如果是的化, 那么键已经过期, 否则的话, 键未过期

过期键的删除策略

* 定时删除
  定时删除策略对内存友好, 通过定时器, 定时删除策略可以保证过期键会尽快的被删除, 并释放内存
  如果过期键较多的情况下, 会占用相当一部分的CPU时间

* 惰性删除
  惰性删除只会在取出键时才对键进行过期检查, 保证删除操作只会在非做不可的情况下进行
  但是如果一直不访问这个键, 那么这个键就会一直存在于内存空间中

* 定期删除
  定期删除策略每隔一段时间执行一次删除过期键操作, 并通过限制删除操作执行的时长和频率减少占用CPU时间

Redis的过期删除策略

* 惰性删除策略
  所有读写数据库的命令, 判断是否过期, 如果过期则删除
* 定期删除策略
  定期并且在规定时间内, 分多次遍历服务器的各个数据库, 从数据库中的过期字典中随机检查一部分键的过期时间, 并删除其中的过期的键

生成RDB文件
  在执行save或bgsave命令创建RDB命令时, 程序会对数据库中的键进行检查, 已过期的键不会被保存到新创建的RDB文件中
载入RDB文件
  主服务器在载入RDB文件时, 程序会对保存的键进行检查, 未过期的键会被载入到数据库中, 过期的键会被忽略
  从服务器在载入RDB文件时, 不会检查键是否过期

AOF文件写入
  如果某个键过期, Redis会向文件中追加一条删除指令, 显示的记录该键被删除
AOF重写
  AOF重写的过程中, Redis会对数据库中的键进行检查, 已过期的键不会被重写到AOF文件中

复制
  当服务器运行在复制模式下, 从服务器的过期键删除动作由主服务器控制

* 主服务器删除一个过期键之后, 会显示的向所有从服务器发送删除指令, 通知从服务器删除这个过期键
* 从服务器在执行客户端发送的读命令时, 即使碰到过期键也不会删除, 而是继续像未过期的键一样处理
* 从服务器只有在接收到主服务器的删除指令才会将键删除

### RDB持久化

Redis 提供了RDB持久化功能, 这个功能可以将Redis在内存中的数据库状态保存到磁盘里面, 避免数据意外丢失

#### RDB文件的创建与载入

* SAVE 命令
  SAVE命令会阻塞Redis服务器进程, 直到RDB文件创建完毕为止
* BGSAVE 命令
  BGSAVE命令会调用fork(), 创建出子进程, 由子进程负责创建RDB文件, 服务器父进程继续处理命令请求

Redis没有主动载入RDB的命令, 所以当服务器检测到存在RDB文件时, 就会自动载入, 在载入期间服务器会一直阻塞, 直到载入完成

#### 自动间隔性保存

通过在配置文件中配置, 类似save 900 1的选项, 满足每隔900秒至少进行1次修改, 则会自动触发BGSAVE命令

#### RDB文件结构

完整RDB文件包含以下五个部分
| REDIS | db_version | databases | EOF | check_sum |

* REDIS部分长度为5字节, 保存着REDIS五个字符, 程序可以在载入文件时, 快速检查是否是RDB文件
* db_version 长度为4字节, 值是一个字符串表示的整数, 整数记录了RDB文件的版本号
* databases部分保存任意多个非空数据库
  每个非空数据库在RDB文件中都可以保存为 | SELECTDB | db_number | key_value_pairs | 三个部分

### AOF持久化

Redis提供了AOF持久化功能, AOF通过记录保存数据库中的写命令来记录数据库状态

#### AOF持久化实现

AOF持久化功能的命令的实现可以分为命令追加, 文件写入, 文件同步三个步骤

* 命令追加
  当AOF持久化功能处于打开状态时, 服务器在执行完一个写命令之后, 会以协议格式将被执行的写命令追加到服务器的状态aof_bf缓冲区末尾
* 文件写入和同步
  服务器在每次结束一个时间循环之前, 它都会调用flushAPpendOnlyFile函数, 判断是否将aof_buf中的内容写入AOF文件里面
  flushAppendOnlyFile函数的行为由appendfsync配置决定

  | appendfsync | flushAppendOnlyFile 行为 |
  |:---:|:---:|
  | always | 将 aof_buf 缓冲区中的所有内容写入并同步到AOF文件 |
  | everysec | 每隔一秒中将AOF文件同步到磁盘中 |
  | no | 同步AOF文件的时机由操作系统决定 |

#### AOF载入

Redis读取AOF文件的步骤如下:

* 创建一个不带网络连接的伪客户端, 因为Redis命令只能在客户端上下文中执行, 而载入AOF文件时使用的命令直接来源于AOF文件
* 从AOF文件中分析并读取一条写命令
* 使用伪客户端执行被读出的写命令
* 重复上两步, 直到文件处理完毕

#### Redis重写

当AOF持久化的内容越来越多, 文件的体积会越来越大, 所以Redis提供了AOF文件重写功能, 创建一个新的AOF文件替换原来AOF文件

Redis会将当前数据库中的所有数据处理成AOF文件中的写命令

* 子进程进行AOF重写期间, 父进程继续处理命令请求
* 子进程带有服务器进程的数据副本, 使用子进程而不是线程, 可以在避免使用锁的情况下, 保证数据安全性

在子进程重写期间, 父进程执行以下三个工作步骤:

1. 执行客户端发来的命令
2. 将执行后的写命令追加到AOF缓冲区
3. 将执行后的写命令追加到AOF重写缓冲区

这样即可保证:

* AOF缓冲区的内容会定期被写入和同步到AOF文件中, 对现有AOF文件的处理工作会如常进行
* 从创建子进程开始, 服务器执行的所有写命令都会被记录到AOF重写缓冲区中

当子进程完成重写任务后, 向父进程发送信号, 父进程接受到信号之后:

1. 将AOF重写缓冲区中的所有内容写入到新AOF文件中, 这是新AOF文件所保存的数据库状态和服务器当前的数据库状态一致
2. 对新的AOF文件进行改名, 原子地覆盖现有的AOF文件, 完成新旧两个AOF文件的替换

整个AOF重写过程中, 只有替换文件时会对服务器进行造成阻塞, 其他时候都不会阻塞父进程, 这将AOF重写的影响降到了最低

### 事件

Redis服务器是一个事件驱动程序:

* 文件事件(file event): 
  Redis服务器通过套接字与客户端进行连接, 而文件事件就是服务器对套接字操作的抽象, 服务器与客户端的通信会产生相应的文件事件
* 时间事件(time event):
  Redis服务器中的一些操作需要在特定事件点执行, 而时间事件就是服务器对这类定时操作的抽象

#### 文件事件

Redis是基于Reactor模式开发了自己的网络事件处理器, 这个处理器被称为文件事件处理器

* 文件事件处理器使用I/O多路复用程序来同时监听多个套接字, 并根据套接字目前执行的任务来为套接字关联不同的事件处理器
* 当被监听的套接字准备好执行连接应答, 读取, 写入, 关闭等操作, 与操作对应的文件事件就会产生, 这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件

文件事件类型

* 当套接字变得可读时或者有新的可应答套接字出现时, 套接字产生AE_READABLE事件
* 当套接字变得可写时, 套接字产生AE_WRITABLE事件

#### 时间事件

时间事件类型:

* 定时事件: 让一段程序在指定时间之后执行一次
* 周期性事件: 让一段程序每个指定时间就执行一次

一个时间事件主要由以下三个属性组成:

* id: 服务器为时间事件创建的全局唯一ID, ID号按顺序递增
* when: 毫秒精度的UNIX时间戳, 记录了时间事件的到达时间
* timeProc: 时间事件处理器其, 当时间事件到达时, 服务器会调用这个函数

### 发布与订阅

Redis的发布与订阅功能由PUBLISH, SUBSCRIBE, PSUBSCRIBE等命令组成

通过执行SUBSCRIBE命令, 客户端可以订阅一个或多个频道, 从而成这些频道的订阅者, 每当有其他客户端向被订阅的频道发送消息时, 频道的所有订阅者都会收到这条消息

#### 频道的订阅与退订

当一个客户端执行SUBSCRIBE命令订阅某个或某些频道的时候, 这个客户端与被订阅频道之间就建立起了一种订阅关系, 服务器会将客户端与被订阅的频道在pubsub_channels字典中进行关联

订阅频道

关联操作:

* 如果频道已经有其他订阅者, 那么它在pubsub_channels字典中必然由相应的订阅者链表, 程序唯一要做的就是将客户端添加到订阅者链表的末尾
* 如果频道还未有任何订阅者, 那么它必然不存在于pubsub_channels字典, 程序首先要在 pubsub_channels字典中为频道创建一个键, 并将这个键的值设置为空链表, 然后再将客户端添加到链表, 成为链表的第一个元素

退订频道

UNSUBSCRIBE命令可以解除客户端与频道之间的关联

* 程序会更具被退订频道的名字, 在pubsub_channels字典中找到频道对应的订阅者链表, 然后从订阅者链表中删除退订客户端的信息
* 如果删除退订客户端之后, 频道的订阅者链表变成了空链表, 那么说明这个频道已经没有任何订阅者了

发送消息

PUBLISH <channel> <message>命令将消息发送给频道

* 将消息message发送给channel频道的所有订阅者
* 如果有一个或多个模式pattern与频道channel匹配, 那么将消息message发送给pattern模式的订阅者

### 事务

Redis 通过 MUTIL, WATCH, EXEC等命令实现事务, 事务将多个命令请求打包, 然后一次性, 按顺序地执行多个命令的机制, 并且在事务执行期间, 服务器不会中断事务而改去执行其他客户端的命令请求, 它会将事务中的所有命令都执行完毕, 然后采取处理其他客户端的命令请求

事务开始
MULTI命令的执行标志着事务的开始, 将执行该命令的客户端切换至事务状态, 这一切换通过在客户端的状态flags属性中打开REDIS_MULTI标识来完成 

命令入队

* 如果客户端发送命令为EXEC, DISCARD, WATCH, MULTI四个命令中的其中一个, 那么服务器立即执行这个命令
* 如果客户端发送的命令其他命令, 那么服务器并不立即执行这个命令, 而是将这个命令放入一个事务队列里面, 然后向客户端返回QUEUED回复

执行事务

当一个处于事务状态的客户端向服务器发送EXEC命令时, 这个EXEC命令将立即被服务器执行, 服务器会遍历该客户端的事务队列, 执行队列中保存的所有命令, 最后将执行命令所得的结果全部返回给客户端

WATCH命令

WATCH命令是一个乐观锁, 他可以在EXEC命令执行前, 监视任意数量的数据库键, 并在EXEC命令执行时, 检查被监视的键是否至少有一个已经被修改过了, 如果是的话, 服务器将拒绝执行事务, 并向客户端返回代表事务执行失败的空回复


