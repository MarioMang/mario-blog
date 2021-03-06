---
title: MySQL(四)
date: '2020-12-19 23:40:00'
category: MySQL
author: MarioMang
cover: /resources/images/default_cover.gif
updated: '2020-12-19 23:40:00'
---

# MySQL(四)

## Scheme与数据类型优化

### 选择优化的数据类型
MySQL 选择数据类型的原则
* 占用字节少的
	> 应该尽量选择正确存储数据的最小数据类型
	> 最小的类型 占用更少的磁盘, 内存和CPU缓存
* 简单的数据类型
	> 简单的数据类型操作需要更少的CPU周期
* 尽量避免NULL
	> 通常情况下, 最好指定列为NOT NULL, 除非真的需要存储NULL
	> 可以为NULL的列, 会使用更多的存储空间, 索引和索引统计和值都比较负责, 当NULL列被索引时, 每个索引记录需要一个额外的字节
	> 如果计划在列上建索引, 尽量避免设计为可NULL的列

#### 整数类型
MySQL整数类型: TINYINT(8bit), SMALLINT(16bit), MEDIUMINT(24bit), INT(32bit), BIGINT(64bit)

MySQL可以指定整数类型的宽度, 它不会限制值的合法范围, 只是规定了客户端显示字符的宽度

MySQL整数进行计算时, 会转为BIGINT进行计算

#### 实数类型
MySQL实数类型: FLOAT(32bit), DOUBLE(64bit), DECIMAL

浮点和DECIMAL类型都可以指定精度, 这会影响列占用的空间, MySQL5.0之后将数字存储在二进制字符串中(每4个字节保存9个数字), 小数点占用一个字节, DECIMAL类型最多允许65个数字

浮点数相比DECIMAL占用更少空间

MySQL实数进行计算时, 会转为DOUBLE类型进行计算

#### 字符串类型

##### VARCHAR
VARCHAR类型用于存储可变长字符串, 是最常见的字符串数据类型, 它比定长类型更节省空间

如果MySQL表使用ROW_FORMAT=FIXED创建的话, 每一行都会使用定长存储

VARCHAR会使用1个或者2个字节保存字符串长度, 如果列的最大长度小于或等于255则使用1个字节保存长度  

适合VARCHAR的情况:
* 字符串列的最大长度比平均长度大很多
* 列的更新很少

##### CHAR
CHAR类型是定长的, MySQL根据定义的字符串长度分配足够的空间, 并且会删除所有末尾的空格

CHAR适合存储很短的字符串, 或者所有的值都接近同一个长度

#### BLOB和TEXT
BLOB和TEXT都是为了存储很大的数据而设计的字符串数据类型, 分别采用二进制和字符方式存储

MySQL把每个BLOB和TEXT值当作一个独立的对象处理, 当值太大时, InnoDB会使用一个外部存储区域来进行存储, 此时每个值在行内需要1-4个字节存储指针, 指向外部存储区域

MySQL只对BLOB和TEXT每个列前的max_sort_length字节做排序, 而不是整个字符串

#### 枚举
MySQL会根据列表值的数量压缩到一个或者两个字节, MySQL在内部会将每个值在列表中的位置保存为整数, 并且在表的.frm文件中保存 数字-字符串 映射关系的查找表

如果数字作为Enum枚举常量, 很容易导致混乱, 所以尽量避免这么做

枚举字段时按照内部存储的整数而不是定义的字符串进行排序的

字符串列表是固定的, 所以添加或删除枚举常量时, 必须使用alter table, 所以可以通过在列表末尾追加元素, 来避免对整张表进行重建


#### 日期和时间
MySQL存储的最小时间粒度是秒

##### DATETIME
从1001年到9999年, 精度为秒, 它把日期封装为YYYYMMDDHHMMSS的整数中, 与时区无关, 使用8个字节的存储空间

##### TIMESTAMP
TIMESTAMP 保存了从1970年1月1日午夜以来的秒数, 它和UNIX时间戳相同, 只有4个字节的存储空间, 依赖时区

#### 位数据类型

##### BIT
可以使用BIT列在一列中存储一个或多个true/false, BIT列的最大长度是64bit

MyISAM会打包存储所有的BIT列, 17bit只需要17bit, Memory和InnoDB, 为每个BIT列使用一个足够存储的最小整数类型

MySQL把BIT当作字符串类型, 而不是数字类型

##### SET
如果需要保存很多的true/false值, 可以考虑合并这些列到一个SET数据类型

#### 选择标识符
* 整数类型
	> 整数通常是标识列最好的选择, 因为他们很快并且可以使用 AUTO_INCREMENT
* ENUM和SET类型
	> ENUM和SET适合存储固定信息或者只包含固定信状态或者类型静态的定义表
* 字符串
	> 如果可能尽量避免使用字符串作为标识类型, 因为他们很消耗空间并且字符串比数字类型慢
	> 	* 因为插入值会随机地写到索引的不同位置, 所以使得INSERT语句变慢, 也许会导致页分裂, 磁盘随机访问, 以及对于聚簇索引产生聚簇索引碎片
	> 	* SELECT语句会变慢, 因为逻辑上相邻的行会分布在磁盘和内存的不同地方
	> 	* 随机值导致缓存对所有类型的查询语句效果都很差
	> 如果使用UUID值, 则应该移除"-"字符, 或者用UNHEX()函数传唤UUID值为16字节的数字  
	> 虽然UUID分布也不均匀, 但还是有一定顺序的, 尽管如此, 但还是不如递增的整数好用

> MySQL Scheme设计中的陷阱
> * 太多的列  
> 		MySQL存储引擎API工作时需要在服务器层和存储引擎层之间通过行缓冲格式拷贝数据, 然后在服务器层将缓冲内容解码成各个列, 从行缓冲中将编码过的列转换成数据结构的操作代价是非常高的
> * 太多的关联  
> 		所谓的 实体-属性-值(EVA) 设计模式是一个常见的糟糕的设计模式, 尤其是在MySQL下不能靠谱的工作, MySQL限制了每个关联操作最多只能有61张表, 在关联少于61张表的情况下, 解析和查询优化的代价也会成为MySQL的问题
> * 全能的枚举  
> 		注意防止过度使用枚举, 因为每次增加枚举值时, 就会需要alter table
> * 变相枚举  
> 		集合SET中允许列中存储一组定义值, 有时候就会比较混乱
> * 非此发明的NULL  
> 		尽量避免使用NULL, 空值尽量使用0值代替, 但是不要走极端, 完全抛弃NULL


### 范式和反范式

对于给定的数据通常都有很多种表示方法, 从完全的范式化到完全的反范式化, 以及两者的折中  
在范式化数据库中, 每个实事数据会出现并且只出现一次, 相反在反范式化数据库中, 信息是冗余的, 可能会出现在多个地方

#### 范式
* 第一范式  
	属性是不可再分割的
* 第二范式  
	表中要有主键, 表中其他字段都依赖主键
* 第三范式  
	表中不能存在其他表中存在的信息相同的字段, 需要使用外键去关联

#### 范式的优点和缺点
* 范式化的更新操作通常比反范式化快
* 范式化的数据, 在修改时需要修改的内容更少
* 范式化的数据表通常更小, 节省空间
* 范式化的数据表, 检索列表数据时, 更少需要DISTINCT或者GROUP BY字段

范式的缺点就是关联, 稍微优点复杂的操作就会需要至少一次关联操作, 这可能造成一些索引失效

#### 混用范式和反范式
事实上, 完全的范式化和完全的反范式化, 通常存在与实验室中, 在实际应用中经常需要混用

### 缓存表和汇总表

有时提升性能最好的方法是在同一张表中保存衍生的冗余数据, 有时也需要创建一张完全独立汇总表或缓存表

汇总表表示存储那些可以比较简单的从scheme其他表中获取数据的表

当重建汇总表和缓存表时, 通常需要保证数据在操作时依然可用, 这就需要使用 影子表 来实现, 当完成了建表操作后, 可以通过一个原子的重命名操作切换影子表和原表

#### 物化视图
物化视图实际上是预先计算并且存储在磁盘上的表, 可以通过各种各样的策略刷新和更新  
类似通过读取 binlog 日志解析相关的变更, 然后重新计算物化视图的内容, 这意味着不需要通过查询原始数据来更新视图

#### 计数器表
如果应用在表中保存计数器, 则在更新计数器时可能碰到并发问题  
创建一张独立的表存储计数器, 这样可以使计数器表小且快

### ALTER TABLE
MySQL 执行修改表操作是用新的结构创建一个空表, 从旧表中查出所有数据插入新表, 然后删除旧表, 这样操作会花费很长时间  
一般而言, 大部分ALTER TABLE操作会导致MySQL服务中断

两种技巧: 
1. 先在一台不提供服务的机器上执行ALTER TABLE操作, 然后和提供服务的主库进行切换
2. 影子拷贝, 用新表结构创建一张和源表无关的新表, 然后通过重命名和删表操作交换两张表

不需要重建表的操作
* 移除一个列的AUTO_INCREMENT属性
* 增加, 移除, 或更改ENUM和SET常量, 如果移除的是已经有行数据用到其值的常量, 查询将会返回一个空值

