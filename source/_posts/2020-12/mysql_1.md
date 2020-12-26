---
title: MySQL(一)
date: '2020-12-19 22:00:00'
category: MySQL
author: MarioMang
cover: /resources/images/default_cover.gif
updated: '2020-12-19 22:00:00'
---

# MySQL(一)

## MySQL 架构与历史

### MySQL逻辑架构

* 第一层, 客户端 处理连接, 授权认证, 安全等等
* 第二层, 服务层 查询解析, 分析, 优化, 缓存以及所有的内置函数, 所有的跨存储引擎功能
* 第三层, 存储引擎 负责MySQL中数据的存储和提取

#### 连接管理与安全性
每个客户端都会在服务器进程中拥有一个线程, 这个连接的查询指回在这个单独的线程中执行  
当客户端连接到MySQL服务器时, 服务器需要对其进行认证, 认证成功后, 继续验证是否具有执行某个特定查询的权限

#### 优化与执行
MySQL会解析查询, 并创建内部数据结构, 然后对其进行各种优化, 包括重写查询, 决定表的读取顺序, 以及选择合适的索引等  
优化器并不关心表使用的是什么存储引擎, 但存储引擎对于优化查询是有影响的

### 并发控制

#### 读写锁
当某个用户在修改某一部分数据时, MySQL 会通过锁定防止其他用户读取同一数据

#### 锁粒度

##### 表锁 table lock
表锁是MySQL中最基本的锁策略, 并且是开销最小的策略, 在加锁时, MySQL会锁定整张表, 一个用户在对表进行写操作时, 需要先获得写锁, 并且阻塞其他用户所有的读写操作

##### 行锁 row lock
行锁可以最大程度的支持并发, 行锁只在存储引擎层实现, MySQL服务层并没有实现

### 事务
事务就是一组原子性的SQL查询, 这组SQL要么全部执行成功, 要么全部执行失败

#### ACID特征
* 原子性(atomicity)
	一个事务必须被视为一个不可分割的最小单元, 整个事务中的所有操作, 要么全部成功, 要么全部失败
* 一致性(consistency)
	数据库总是从一个一致性状态转换到另一个一致性状态
* 隔离性(isolation)
	一个事务所做的修改在最终提交之前, 对其他事务是不可见的
* 持久性(durability)
	一旦事务提交, 事务所做的修改就会被持久化到数据库中

#### 隔离级别
SQL标准中定义了4中隔离级别
* READ UNCOMMITED 读未提交
	事务中的修改, 即使没有提交, 对其他事务也是可见的, 其他事务可以读取未提交的数据, 被称为脏读(Dirty Read)
* READ COMMITED 读已提交
	事务开始时, 只能看见自己已经提交的事务所做的修改, 在事务提交前, 对其他事务都是不可见的, 但是两次执行同样的查询, 可能会得到不一样的结果
* REPEATABLE READ 可重复读
	保证了同一个事务中多次读取同样的记录结果是一致的, 同样无法解决幻读的问题
* SERIALIZABLE 序列化
	通过强制事务串行化, 避免幻读的问题, 该级别会在读取的每一行数据上都加锁, 可能导致大量的超时和抢占锁的问题

|隔离级别|脏读|不可重复读| 幻读 | 加锁读 |
|:--:|:---:|:---:|:---:|:---:|
|READ UNCOMMITED| true| true | true |false|
|READ COMMITED | false | true | true | false |
| REPEATABLE READ | false | false | true | false |
| SERIALIZABLE | false | false | false | true| 

#### 死锁
死锁指多个事务在同一资源上互相占用, 并请求锁定对方占用的资源, 导致事务之间互相阻塞, 无法继续进行

#### 事务日志
事务日志可以帮助提高事务的效率, 其表现为, 存储引擎在修改表的数据时, 只需要修改其内存拷贝, 再把该修改行为记录到持久在硬盘上的事务日志中, 而不用每次都将修改的数据本身持久到硬盘  
事务日志采用的是追加的方式, 因此写日志的操作是磁盘上一小块区域内的顺序IO, 相比于随机IO来说要快很多  
这种方式通常称为预写式日志(WriteAheadLogging), 修改数据需要写两次磁盘

#### MySQL 中的事务
MySQL提供了两种事务型的存储引擎: InnoDB 和 NDB Cluster

##### 自动提交(AUTO COMMIT)
MySQL默认采用自动提交的模式, 如果不是显示的声明开始事务, 则每个查询都被当作一个事务执行提交操作

可以通过下面的语句, 查看是否启用来自动提交模式: 
``` SQL
SHOW VARIABLES LIKE 'AUTOCOMMIT';
``` 

##### 隐式和显示锁定
InnoDB采用的是两阶段锁定协议, 在事务执行的过程中, 随时都可以执行锁定, 锁只有在执行 COMMIT 或者 ROLLBACK 时才会释放, 并且所有锁是同一时间释放的

Inno支持通过特定语句进行显示锁定
``` SQL
SELECT ... LOCK IN SHARE MODE;
SELECT ... FOR UPDATE;
```

### 多版本并发控制
基于提升并发性能的考虑, 大多数事务型存储引擎都实现了多版本并发控制(MVCC), 典型的有乐观并发控制和悲观并发控制  
InnoDB 的MVCC, 是通过在每行记录后面保存两个隐藏列实现的, 一个保存了行的创建时间, 一个保存了行的过期时间, 实际上是系统版本号, 每开始一个事务, 版本号都会递增, 事务开始时的版本号会被作为事务的版本号, 用来和查询到的每行记录版本号进行比较

在REPEATABLE READ下 MVCC具体是如何操作的
* SELECT
	> InnoDB 会根据以下两个条件检查每行记录:  
	* InnoDB 只查找版本早于当前事务版本的数据行, 这样可以确保事务读取的行, 要么是在事务开始前已经存在的, 要么是事务自身插入或者修改的
	* 行的删除版本要么未定义, 要么大于当前事务版本号，这样可以确保事务读取到的行, 在事务开始之前未被删除
* INSERT 
	> InnoDB 为新插入的每一行保存当前系统版本号作为行版本号
* DELETE 
	> InnoDB 为删除的每一行保存当前系统版本号作为删除标识
* UPDATE
	> InnoDB 为插入一行新纪录, 保存当前系统版本号作为行版本号, 同时保存当前系统版本号到原来的行作为删除标识

MVCC只在REPEATABLE READ 和 READ COMMITED两个隔离级别下工作, 因为READ UNCOMMITED 总是读取最新的数据行, 而不是符合当前事务版本的数据行, SERIALIZABLE则会对所有读取的行都加锁

### MySQL的存储引擎
MySQL将每个数据库保存为数据目录下的一个子目录, 创建表时, MySQL 会在数据库子目录下创建一个和表同名的.frm文件保存表的定义

可以使用下面语句来查看表的相关信息
``` SQL
show table status like 'user';
```
下面写了每一行的含义
* Name: 表名
* Engine: 表的存储引擎
* Row_format: 行的格式 Dynamic, Fixed, Compressed
* Rows: 表中的行数, 对于MyISAM, 该值是精确的
* Avg_row_length: 平均每行包含的字节数
* Data_length: 表数据的大小
* Max_data_length: 表数据的最大容量, 该值和存储引擎有关
* Index_length: 索引大小
* Data_free: 对于MySIAM表, 表示已分配但是目前没有使用的空间
* Auto_increment: 下一个AUTO_INCREMENT的值
* Create_time: 表的创建时间
* Update_time: 表数据的最后修改时间
* Check_time: 使用CHECK TABLE命令或者myisamchk工具最后一次检查表的时间
* Collation: 表的默认字符集和字符列排序规则
* Checksum: 如果启用, 保存的是整个表的实时校验和
* Create_options: 创建表时指定的其他选项
* Comment: 该列包含了一些其他的额外信息

#### InnoDB 存储引擎
InnoDB 是MySQL的默认事务型引擎, 它被设计为用来处理大量的短期事务, 短期事务大部分情况是正常提交的

InnoDB 的数据存储在表空间中, 表空间是由InnoDB管理的一个黑盒子, 由一系列的数据文件组成, InnoDB 采用MVCC来支持高并发, 并且实现了四个标准的隔离级别, 其默认级别是 REPEATABLE READ, 并且通过间隙锁策略防止幻读的出现, 间隙锁使得InnoDB不仅仅锁定查询设计的行, 还会对索引中的间隙进行锁定, 以防止幻影行的插入

InnoDB表是基于聚簇索引建立的, 聚簇索引对主键查询有很高的性能, 不过它的二级索引中必须包含主键列

#### MyISAM存储引擎
MyISAM提供了大量的特性, 包括全文索引, 压缩, 空间函数(GIS)等, 但是它不支持事务和行级锁, 并且有一个缺陷就是崩溃后无法安全恢复

MyISAM特性: 
* 加锁与并发
	> MyISAM对整张表加锁, 而不是针对行
* 修复
	> 对于MyISAM, MySQL 可以手工或者自动执行检查和修复操作
* 索引特性
	> 对于MyISAM表, 即使是BLOB和TEXT等长字段, 也可以基于其前500个字符创建索引
* 延迟更新索引键
	> 创建MyISAM表时, 如果指定了DELAY_KEY_WRITE选项, 在每次修改执行完成时, 不会立刻将修改的索引数据写入磁盘, 而是会写到内存中的键缓冲区, 只有在清理键缓冲区或者关闭表的时候才会将对应的索引块写入到磁盘

MyISAM 压缩表  
如果表在创建并导入数据以后, 不会再进行修改操作, 那么这样的表或许适合采用MyISAM压缩表


#### MySQL其他存储引擎

* Archive 引擎
	> Archive 存储引擎只支持INSERT和SELECT操作, 会缓存所有写并利用zlib对插入的行进行压缩  
	> Archive 支持行锁和专用的缓冲区, 所以可以实现高并发的插入

* Blackhole 引擎
	> Blackhole 引擎没有实现任何的存储机制, 它会丢弃所有插入的数据, 不做任何保存操作, 但是服务器会记录Blackhole的日志, 所以可以用于复制数据到备库

* CSV 引擎
	> CSV引擎可以将普通的CSV文件作为MySQL的表来处理, 但是这种表不支持索引

* Federated 引擎
	> Federated 引擎是访问其他MySQL服务器的代理, 他会创建一个到远程MySQL服务器的客户端连接, 并将查询传输到远程服务器执行, 然后提取或者发送需要的数据

* Memory 引擎
	> Memory表会把数据都保存在内存中, 不需要进行磁盘I/O, Memory表的结构在重启以后会保留, 但是数据会丢失
	> Memory的场景:
	> 	* 用于查找或者映射表
	> 	* 用于缓存周期性聚合数据的结果
	> 	* 用于保存数据分析中产生的中间数据
	> Memory 表支持Hash索引, 因此查找操作非常快

* Merge 引擎
	> Merge 引擎是 由多个 MyISAM表合并而来的虚拟表
	> 适合用于日志或者数据仓库

* NDB集群引擎
	