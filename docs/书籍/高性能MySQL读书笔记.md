# 第1章 MySQL架构与历史

MySQL最重要、最与众不同的特性是它的存储引擎架构，这种架构将查询处理与数据的存储/提取相分离，使得可以在使用时根据不同的需求来选择数据存储的方式。

## MySQL逻辑架构

![image-20201020013258450](../images/image-20201020013258450.png)

服务器通过API与存储引擎进行通信，这些API屏蔽了不同存储引擎之间的差异。存储引擎API包含几十个底层函数，不同存储引擎不会去解析SQL，而只是简单地响应上层服务器的请求。

### 连接管理与安全性

- 每个客户端连接都会在服务器进程中拥有一个线程，这个连接的查询只会在这个线程中执行，该线程只能轮流在某个CPU核心中运行。MySQL基于线程池来管理线程（创建、缓存、销毁）。
- 当客户端连接到MySQL服务器时，服务器会基于用户名、主机信息、密码等进行认证（SSL）。一旦客户端连接成功，服务器会继续验证该客户端是否具有执行某个特定查询的权限。

### 优化与执行

- MySQL会解析查询并创建查询解析树，然后对其进行各种优化，如重写查询、决定表的读取顺序、选择合适的索引等。用户可以通过使用关键字提示优化器，从而影响它的决策过程。explain解释优化过程各个因素
- 优化器不关心表使用的是什么存储引擎，但存储引擎对于优化查询是有影响的：优化器会请求存储引擎提供容量或某个具体操作的开销信息，以及表数据的统计信息等。  例如索引
- 对于SELECT语句，在解析查询之前，服务器会先检查查询缓存，如果找到对应的查询，就不必再执行查询解析、优化和执行的整个过程，而是直接返回查询缓存中的结果集。

## 并发控制

### 读写锁

处理并发读或者写时可以使用共享锁 和 排他锁，也叫读锁和写锁

### 锁粒度

加锁本身也需要消耗资源，锁策略就是在锁的开销和安全性之间寻求平衡。每种MySQL存储引擎都可以实现自己的锁策略和锁粒度。

`表锁`：

开销最小的策略

会锁定整张表，在对表进行写操作之前，需要先获得写锁，获得写锁后将会阻塞其他用户对该表的读写操作。只有没有写锁时，其他用户才能获取读锁，读锁之间是不相互阻塞的。写锁比读锁有更高的优先级，因此一个写锁请求可能会被插入到读锁队列的前面。

虽然不同的存储引擎都有自己的锁实现，MySQL自身仍然会在服务器层使用表锁并忽略存储引擎的锁机制，例如当执行ALTER TABLE时，`服务器`会使用表锁。

`行级锁`：

可以最大程度程度地支持并发处理（同时带来了最大的锁开销）

行级锁只在存储引擎层实现，MySQL服务器层没有实现。

## 事务

ACID：atomicity(原子性)、consistency(一致性)、isolation(隔离性)、durability(持久性)。

![image-20201017233832529](../images/image-20201017233832529.png)

事务的ACID特性可以确保银行不会丢你的钱

### 隔离级别

隔离级别：SQL标准定义了4种隔离级别，每一种级别都规定了在一个事务中所做的修改，哪些在事务内和事务间是可见的，哪些是不可见的。  较低的隔离级别通常可以执行更高的并发，系统的开销也更低。

1. `READ UNCOMMITED` 事务中的修改，即使没有提交，对其他事务也是可见的。（因此会产生脏读）
2. `READ COMMITTED` 一个事务只能看见已经提交的事务所做的修改。
3. `REPEATABLE READ` 这是MySQL默认的事务隔离级别，保证在同一个事务中多次读取同样记录的结果是一样的。理论上该级别无法避免幻读的问题，InnoDB通过多版本并发控制解决了幻读的问题。
4. `SERIALIZABLE` 强制事务串行执行，会在读取的每一行数据上都加锁，可能导致大量的超时和锁争用问题。

可以在配置文件中设置整个数据库的隔离级别，也可以只改变当前会话的隔离级别： SET SESSION TRANSACTION LEVEL READ COMMITTED;

![image-20201018130009678](../images/image-20201018130009678.png)

### 死锁 

**两个或者多个事务在同一资源上相互占用，并请求锁定对方占用的资源，从而导致恶性循环**

当多个事务试图以不同的顺序锁定资源时，就可能产生死锁。   多个事务同时锁定同一资源也会产生死锁。

**为了解决问题，数据库系统实现了各种死锁检测和死锁超时机制**

对于事务型的系统，死锁发生后，只有部分或者完全回滚其中一个事务，才能打破死锁。InnoDB目前处理死锁的方式是：**在检测到死锁循环依赖后，将持有最少行级排它锁的事务进行回滚。**

处理死锁，只要重新执行死锁回滚的事务即可。

### 事务日志

事务日志：事务日志可以帮助提高事务的效率，存储引擎在修改表的数据时只需要修改表数据的内存拷贝，同时把该修改行为持久化到硬盘中的事务日志中，相比于将修改的数据本身持久化到磁盘，事务日志采用的是追加的方式，因此是在磁盘上的一小块区域内顺序地写入，而不是随机的I/O操作。事务日志持久化后，内存中被修改的数据在后台可以慢慢刷回磁盘，如果在数据没有写回磁盘时系统崩溃了，存储引擎在重启时能够自动恢复这部分数据。目前大多数存储引擎都是这样实现的。

**我们称之为预写式日志（Write-Ahead Logging），修改数据需要写两次磁盘。**

### MySQL中的事务

#### 自动提交

MySQL默认采用自动提交模式，如果不是显式地开始一个事务，则每个查询都会被当做一个事务执行提交操作。可以在当前连接中设置`AUTOCOMMIT`变量来禁用自动提交模式(禁用后，需要显式地执行COMMIT提交或者ROLLBACK回滚)。

对于非事务型的表，修改AUTOCOMMIT不会有任何影响，这类表相当于一直处于AUTOCOMMIT启用的状态。 此外，有一些命令，例如ALTER TABLE，在执行之前会强制执行COMMIT提交当前的活动事务。

#### 事务中混合使用存储引擎

如果在事务中混合使用了事务型和非事务型的表，当事务需要回滚时，非事务型表上的变更将无法撤销，这将导致数据库处于不一致的状态。在非事务型表上执行事务相关操作时，MySQL通常并不会报错，只会给出一些警告。

#### 隐式和显式锁定

**InnoDB采用两阶段提交协议**

显式锁定：

InnoDB也支持通过特定的语句进行显式锁定，这些语句不属于SQL规范，如： SELECT … LOCK IN SHARE MODE SELECT … FOR UPDATE

MySQL也支持LOCK TABLES和UNLOCK TABLES语句，这是在服务器层实现的，但它们不能代替事务处理，如果应用到事务，还是应该选择事务型存储引擎。 （建议：除了在事务中禁用了AUTOCOMMIT时可以使用LOCK TABLE之外，其他任何时候都不要显式地执行LOCK TABLES，不管使用的是什么存储引擎，因为LOCK TABLE和事务之间相互影响时，问题会变得非常复杂）

## 多版本并发控制

MYSQL的大多数事务型存储引擎实现都不是简单的行级锁。基于并发性能的考虑，一般都同时实现了多版本并发控制

可以认为MVCC是行级锁的一个变种，但是它在很多情况下都避免了加锁操作，因此开销更低（更高并发）。

其实现原理是通过保存数据在某个时间点的快照来实现的，根据事务开始的时间不同，每个事务对同一张表，同一时刻看到的数据可能是不一样的。

 InnoDB的MVCC是通过在每行记录后面保存两个隐藏的列来实现的，这两个列一个保存行的创建时间，一个保存行的过期时间（或删除时间），当然并不是实际的时间值，而是系统版本号。每开始一个新的事务，系统版本号就会自动递增。事务开始时刻的版本号会作为事务的版本号用来和查询到的每行记录的版本号进行比较。 MVCC只在REPEATABLE和READ COMMITED两个隔离级别下工作，其他两个隔离级别和MVCC不兼容。

![image-20201019181917467](../images/image-20201019181917467.png)

## MySQL的存储引擎

### InnoDB

InnoDB是MySQL默认的事务型引擎。除非有非常特别的原因需要使用其他的存储引擎，否则应该优先考虑InnoDB引擎。

InnoDB的数据存储在表空间中，表空间是由InnoDB管理的一个黑盒子，有一系列的数据文件组成。InnoDB采用MVCC来支持高并发，默认隔离级别是可重复读，并且通过间隙锁策略防止幻读的出现。间隙锁使得InnoDB不仅锁定查询涉及的行，还会对索引中的间隙进行锁定，以防止幻影行的插入。

InnoDB表是基于聚簇索引建立的，聚簇索引对主键查询有很高的性能，但是二级索引（非主键索引）中必须包含主键列。

### MyISAM

MyISAM提供了全文索引、压缩、空间函数等，但是不支持事务和行级锁，且崩溃后数据无法恢复。最典型的问题还是锁表问题。

MyISAM会将表数据存储在两个文件中：数据文件和索引文件。会将数据写到内存中，然后定期将数据刷到磁盘上。

MyISAM对整张表加锁，读取时对表加共享锁，写入时对表加排他锁。支持并发插入（读取的同时，进行写入）。

压缩表：表创建并导入数据后不再进行修改，可以使用压缩表。极大的减少磁盘空间占用，也就减少了磁盘I/O，从而提升查询性能。

### Archive

适合日志和数据采集

### CSV

可以作为数据交换机制

### Memory

如果需要快速访问数据，且数据不会被修改，重启后数据丢失也没关系（因为是存储在内存中的），则可以使用。会比MyISAM快至少一个数据量。重启后数据会丢失，但是表结构会保留。可以用来缓存周期性的聚合数据或保存数据分析中产生的中间数据。

临时表只在单个连接中可见，连接断开时，临时表也将不存在。所以临时表和memory表是不一样的。

### NDB集群引擎

MySQL集群

### 选择存储引擎时的考虑因素

事务、备份、崩溃恢复、特有的特性

**MySQL拥有分层架构，上层是服务器的服务和查询执行引擎，下层则是存储引擎。在存储引擎和服务层之间处理查询是通过API来回交互。对于InnoDB来说，所有操作都是事务。**

# 第2章 MySQL基准测试

基准测试是针对系统设计的一种压力测试，通常的目标是为了`掌握系统的行为`。

## 基准测试的策略

基准测试主要两种策略：集成式（针对整个系统的整体测试）、单组件式（单独测试MySQL）

`测试指标`：

1. 吞吐量：单位时间内的事务处理数；
2. 响应时间（延迟）：完成测试任务所需的时间；（95%法则）
3. 并发性：测试应用在不同并发下的性能；
4. 可扩展性：线性扩展；

影响测试结果的因素

- 数据量
- 数据分布
- 系统预热

文档输出很重要

先收集所有的原始数据，然后再基于此做分析和过滤是一个好习惯

## 基准测试方法

避免常见错误
设计和规划基准测试
基准测试的运行时间设置
获取系统的性能和状态
收集分析绘图

## 基准测试工具

集成式测试工具

1. ab: Apache http服务器基准测试工具,测试服务器qps, 针对单个url进行尽可能快的压力测试
2. http_load: 对Web服务器,可以通过输入文件对多个url进行随机测试
3. JMeter: 测试Web应用和其他入FTP服务器或通过JDBC进行数据库查询测试

单组件式测试工具

1. mysqlslap: 模拟服务器负载并输出计时信息
2. sqlbench: 测试服务器执行查询的速度
3. super smack: mysql 和postgresql的基准测试工具
4. database test suite
5. sysbench: 多线程系统压测工具, 支持mysql,操作系统,和硬件测试

# 第3章 服务器性能剖析

> 如何确定系统是否达到最佳的状态？
>
> 某条语句为什么执行不够快？
>
> 为什么会发生停顿、堆积或者卡死的某些间歇性故障？

性能：完成某件任务所需要的时间度量，性能即响应时间。 性能剖析（profiling）：测量服务器的时间花费在哪里。 性能剖析的步骤：测量任务所花费的时间，然后对结果进行统计和排序，将重要的任务排到前面。

## 剖析MySQL查询

1. SHOW PROFILE: 将profiling开启后,执行的sql的剖析信息会被记录到临时表, 通过 show profile from query (index)命令即可以获取到执行的过程信息
2. SHOW STATUS: 非剖析工具,主要用在对于执行完后观察某些计数器的值
3. 使用慢查询日志

## 诊断间歇性问题

间歇性的问题比如系统偶尔停顿或者慢查询，很难诊断。

1. show global status: 通过某些计数器的"尖刺"或"凹陷"分析问题
2. show processlist: 观察是否有大量线程处于不正常的状态或其他不正常特征
3. 使用innotop: 实时地展示服务器正在发生的事情，监控innodb,监控多个MySQL实例
4. 使用慢查询日志

## 其他剖析工具

1. 自带的一些INFORMATION_SCHEMA统计表
2. 使用strace工具,pt-iopfofile工具生成I/O活动报告

# 第4章 Schema与数据类型优化

## 选择优化的数据类型

**选择合适数据类型的三个原则**

- 更小的通常更好 - 速度更快，占用更少
- 简单就好 - 简单数据类型占用更少的CPU周期，例如整型的比字符串操作代价更低
- 尽量避免NULL - 查询包含NULL的列，对Mysql来说更难优化，因为会使得索引，索引统计和值比较更为复杂

### 整数类型

整数的类型有：TINYINT 、SMALLINT、MEDIUMINT、INT、BIGINT，分别使用8，16，24，32，64位存储空间

它们存储的值的范围：-2（N-1） 到  2（N-1）- 1

其中N为存储空间的位数

Mysql可以为整数类型指定宽度，例如INT(11)，对大多数应用这是没有意义的，它不会限制值的合法范围，只是规定了一些交互工

具用来显示字符的个数而已，对于存储和计算来说，INT(1) 和 INT(20) 是相同的

### 实数类型

在mysql的数据类型中浮点型分为两种，float()与double()类型，定点型为decimal()

数据类型(M,D)  -》M：精度，数据的总长度；  D：标度，小数点后的长度；

其区别在于：

- 当不指定精度时，Float、Double默认会保存实际精度，而Decimal默认是整数
- 当标度不够时，都会四舍五入，但Decimal会警告信息

### 字符串类型

VARCHAR 和 CHAR是最主要的字符串类型，CHAR自不必说，实际使用的情况较少，例如存储男/女，YES/NO等确定长度的字符串，但是这种固定的情况有时候用整型去存储效率更高，所以视情况而定吧

VARCHAR存储的是可变长字符串，它比定长类型更节省空间，因为它仅使用必要的空间，它需要用1个或者2个额外字节记录字符串长度

BLOB和TEXT类型：

- BLOB：二进制存储，没有排序规则和字符集
- TEXT：字符串存储，有排序规则和字符集

> 当BLOB和TEXT值太大时，InnoDB会使用专门的“外部”存储区域来进行存储，此时每个值在行内都需要1~4个字节存储一个指针，然后在外部存储区域存储实际的值

### 日期和时间类型

对于日期和时间类型，据我了解到身边的人大多都不会把时间直接存储到数据库中，同时《高性能Mysql》一书中也推荐另一种做法去存储时间，在这里推荐一下，即：

通过BIGINT类型存储毫秒/微秒级别的时间戳，再显示或者计算的时候都基于时间戳进行计算

### 选择标识符

选择标识列（identifier column）类型时，不仅要考虑存储类型还要考虑如何进行计算和比较，一旦选定了一种类型，还要确保所

有关联表中使用同样的类型，类型之间需要精确匹配（包括UNSIGNED这样的属性）

整数通常是ID列最好的选择。

使用MD5(),SHA1(),UUID()产生的字符串的值会随机分布在很大的空间中，导致INSERT和一些SELECT语句变得很慢：

- 插入值随机写到索引的不同位置，导致页分裂，磁盘随机访问等，详见第五章
- 逻辑上相邻的行会分布在磁盘和内存的不同地方
- 随机值使得缓存赖以工作的`访问局部性原理`失效。

存储UUID值应该移除“-”符号；最好使用UNHEX()函数将UUID值转换为16字节的数字，存储在BINARY(16)列中。检索时可以通过HEX()函数格式化成十六进制格式

### 特殊类型数据

例如：IPV4地址，人们经常使用 VARCHAR(15)列来存储IP地址。然而,它们实际上是32位无符号整数,不是字符串。用小数点将地址分

成四段的表示方法只是为了让人们阅读容易。所以应该用无符号整数存储IP地址，MYSQL提供INET_ATON()和 INET NTOA()函数在这

两种表示方法之间转换

## MySQL schema设计中的陷阱

我们应该避免以下几种情况的出现：

- 太多的列     存储引擎API需要在服务器层和存储引擎层之间通过行缓冲格式拷贝数据，然后在服务器层将缓冲内容解码成各个列，这个操作的代价非常高，而转换的代价依赖于列的数量。
- 太多的关联（上限61张表） 单个查询做好12个表以内关联
- 全能的枚举，变相的枚举
- 随随便便的NULL

## 范式和反范式

反范式的标志：信息冗余

随着时代和机器的发展，我们会经常使用空间换时间的策略，因此基本淘汰了完全遵循范式的做法，但是在范式与反范式中间一定

要根据业务需求做好设计，减少不必要的空间浪费

## 缓存表和汇总表

缓存表：存储那些可以比较简单地从schema其他表获取数据的表（存在逻辑上冗余的数据）； 汇总表：保存使用GROUP BY语句聚合数据的表；

利用Mysql做缓存的可能很少，用作汇总表的可能很多，提供原文中两个场景的较好的解决方案：

**不严格技术或小范围计数** 

```
例如，如何更好的汇总一天中任务执行次数?
```

我们可以采用分割的思想，把一天划成小时，一天过去进行数据汇总时全部累加即可

### 计数表

```
例如，如果统计网站访问人数更合适?
```

我们当然可以用一条记录总人数，也可以用N天（条）数据记录总人数，然后累加，但是Mysql在执行时候有一定的延时，可能一秒

之内有好几十个人点击，那我们可以针对一条数据进行分割成1-100条数据，通过算法求余等等，让100条数据可以均匀的一起工作   （避免锁竞争）

这样可以大幅度增加效率，最终再汇总即可

## 加快ALTER TABLE操作的速度

常规的方法是建另一张结构符合要求的表，并插入旧表的数据，然后切换主从，或者切换新旧表。 修改表的.frm文件是很快的，因此可以为想要创建的表结构创建一个新的.frm文件，然后用它替换掉已经存在的表的.frm文件。（详略）



不知道其他公司对于底层数据库的字段是否会经常调整，反正我们公司每次需求都会涉及数据库字段的调整，每次都需要执行

ALTER操作，如何提高效率？

- 主从库切换，减少ALTER影响时间
- “影子拷贝”，完全创建新表，然后通过重命名+删除的方式替换旧表（无数据的情况）

## 在有索引的情况下快速导入表数据

常规的方法：

1. 先删除索引：如ALTER TABLE t.data DISABLE KEYS; # 对唯一索引无效
2. 导入数据
3. 再创建索引：如ALTER TABLE t.data ENABLE KEYS;

hack方法是直接修改.MYD、.frm、.MYI等文件，详略。

# 第5章 创建高性能索引

## 索引基础

在MySQL中，索引是在存储引擎层而不是服务器层实现的，并没有统一的索引标准，不同的存储引擎索引工作的方式是不一样的。

## 索引类型

1.B-Tree 索引 ： 可用于全值匹配、最左前缀匹配、列前缀匹配、范围值匹配、精确匹配某一列并范围匹配另外一列、只访问索引的查询 

- ![image-20201020032557303](../images/image-20201020032557303.png

B-Tree索引的限制：

- （1）如果不是按照索引的最左列开始查找，则无法使用索引；
- （2）不能跳过索引中的列；
- （3）如果查询中有某个列的范围查询（包括LIKE等），则其右边所有列都无法使用索引优化查找；



`2.哈希索引` 基于哈希表实现，对于每一行数据存储引擎都会对所有的索引列计算一个哈希码，只有精确匹配索引所有列的查询才有效。 只有Memory引擎显式支持哈希索引。（详略）

InnoDB有一个特殊的功能：`自适应哈希索引`，当InnoDB注意到某些索引值被使用得非常频繁时，就会在内存中基于B-Tree索引之上再创建一个哈希索引，这样就让B-Tree索引也具有哈希索引的一些优点，比如快速的哈希查找。

3.空间数据索引（R-Tree） MyISAM表支持空间索引，可以用作地理数据存储。

4.全文索引 查找的是文本中的关键词，而不是直接比较索引中的值。适用于MATCH AGAINST操作，而不是普通的WHERE条件操作。

## 对于Mysql索引使用的歧义

对于很早的版本，一直强调：最左前缀匹配，索引使用顺序，不可跳过索引中的列等等，但高性能Mysql一书之后的版本中对于

mysql优化器进行了很大程度的优化，使得开发者在使用时候可以不用顾虑太多

例如：

```
CREATE TABLE `mytest`.`student`  (
  `age` int(11) NULL DEFAULT NULL COMMENT 'age',
  `name` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT 'name',
  `year` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT 'year',
  INDEX `demo`(`age`, `name`, `year`) USING BTREE COMMENT 'demo'
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;
 
```

故意使用不按索引顺序查询：

```
mysql> explain extended  SELECT * FROM `student` where `name` = 1 and `age` = 1;
实际: key:demo 使用到了索引
 
```

`如何查看Mysql优化之后的SQL`：

```
# 仅在服务器环境下
explain extended  SELECT * FROM `student` where `name` = 1 and `age` = 1;

# 再执行
show warnings;

# 结果如下：
/* select#1 */ select `mytest`.`student`.`age` AS `age`,`mytest`.`student`.`name` AS `name`,`mytest`.`student`.`year` AS `year` from `mytest`.`student` where ((`mytest`.`student`.`age` = 1) and (`mytest`.`student`.`name` = 1))
 
```

可以发现真正执行的SQL是 age在前，name在后

对于仅使用year字段的查询结果呢？依然显示 -> key:demo 使用到了索引

总结：

- 写代码时对于索引顺序不必有`太多顾虑`
- 优化SQL时候不仅仅只看 key参数是否使用索引，还需要注意type，ref等等
- Mysql 优化器`并不是一定改变SQL的参数查询顺序`，而是以它的判定方式选择它认为的最有效的查询



## 索引的优点

正常的索引数据结构可以有其独特的优点，比如哈希索引不支持顺序查询，而B-Tree，或者 B+Tree则可以，总结B-Tree系列索引优点

1. 减少服务器需要扫描的数据量；
2. 避免排序和临时表；
3. 将随机I/O变为顺序I/O；

## 高性能的索引策略

### 独立的列

`索引在使用中不可嵌入表达式或者方法` 错误示范： select * from where eid + 1 < 10;

### 前缀索引和索引的选择性

如果索引很长的字符串列，会导致索引变得很大并且很慢。一个处理策略是模拟哈希索引，另一个办法是索引开始的一部分字符串的值，即前缀索引。

**索引的选择性**指不重复的索引值和数据表中记录总数(#T)的比值。取值在为(1/ #T~1]。选择性越高则查询效率越高。唯一索引的选择性是1，是最好的索引选择性，性能也是最好的。

前缀索引的**优点**是可以节约索引空间，提高索引效率。**缺点**是降低了索引的选择性，同时也无法使用前缀索引做ORDER BY 或 GROUP BY 操作，无法使用前缀索引做覆盖扫描。

使用前缀索引时，**确定前缀长度的依据**是：计算前缀索引的选择性，使前缀索引选择性接近完整列的选择性

### 多列索引

错误做法：为where字段的每一个列创建独立的索引，或者按照错误的顺序创建多列索引

在explain中，如果发现索引合并，实际上说明了表上的索引建的不够好：

- 当需要进行AND操作时，其实说明了需要建立一个包含所有相关列的多列索引；
- 当需要进行OR操作时，通常需要耗费大量CPU和内存资源在算法的缓存、排序和合并操作上；
- 优化器不会把这些计算到”查询成本“中，这会使得查询成本被低估，可能还不如全表扫描然后union。

### 选择合适的索引顺序

不考虑排序和分组时：将选择性最高的列放在前面通常是很好的（选择性即有效区分数据）

但，性能不只是依赖于所有索引列的选择性，也和查询条件的具体值有关，也就是和值的分布有关，这就意味着可能需要

根据那些运行频率最高的查询来调整索引列的顺序

例如：一张订单表，需要查询的字段有两个，一个是地区ID，一个是用户ID，简单思考就一定能发现用户ID的区分度应该是比地区

ID要高的多（相同用户ID的数据要远远少于相同地区ID），因此建立索引应该是（用户ID，地区ID）

### **聚簇索引**

**聚簇索引：**将数据存储与索引放到了一块，索引结构的叶子节点保存了行数据

**非聚簇索引：**将数据与索引分开存储，索引结构的叶子节点指向了数据对应的位置

**在innodb中**，在聚簇索引之上创建的索引称之为辅助索引，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引。**辅助**

**索引叶子节点存储的不再是行的物理位置，而是主键值，辅助索引访问数据总是需要二次查找**

**聚簇索引具有唯一性**，由于聚簇索引是将数据跟索引结构放到一块，因此一个表仅有一个聚簇索引， **聚簇索引默认是主键**，

如果表中没有定义主键，InnoDB 会选择一个**唯一且非空的索引**代替。如果没有这样的索引，InnoDB 会**隐式定义一个主键（类似**

**oracle中的RowId）**来作为聚簇索引。如果已经设置了主键为聚簇索引又希望再单独设置聚簇索引，必须先删除主键，然后添

加我们想要的聚簇索引，最后恢复设置主键即可

聚簇的数据的优点：

- 可以把相关的数据保存在一起。如电子邮箱中，根据用户ID来聚簇数据，这样只要读取少数的数据页就可以获取某个用户的全部邮件。如果没有使用聚簇索引，可能每一封邮件导致一次磁盘I/O；
- 访问数据更快。聚簇索引将索引和数据报错在同一个B-Tree中，因此获取数据通常比非聚簇索引更快；
- 使用覆盖索引扫描的查询可以直接使用页节点的主键值

聚簇索引的缺点：

- 聚簇索引能够提高I/O的密集程度，但如果所有的数据全都放在内存中，那么访问顺序就没那么重要了，聚簇索引也就没什么优势了
- 插入速度严重依赖于插入顺序。按照主键顺序插入是最快的。如果不是先找主键顺序加载数据，那么加载完成后最好用OPTIMIZE TABLE命令重新组织一下表
- 更新聚簇索引列的代价很高
- 基于聚簇索引的表在插入新行，或者主键被更新导致需要移动行时，可能面临"页分裂"的问题
- 举措索引可能导致全表扫描变慢，尤其是行比较稀疏，或者页分裂导致数据存储不连续的时候
- 二级索引（非聚簇索引）可能比想象的大，因为二级索引的叶子节点包含了引用行的主键列
- 二级索引访问需要两次索引查找，而不是一次（第一次获得主键，第二次根据主键值去聚簇索引中查找对应的行）

#### 在InnoDB表中按主键顺序插入行

如果正在使用InnoDB表并且没有什么数据需要聚集，那么可以定义一个代理健作为主键。最简单的是使用AUTO INCREMENT自增

列，这样可以保证数据行是顺序写入的，对于主键做关联操作也是最好的

最好避免随机的（不连续且值的分布范围非常大）聚簇索引，特别是对于I/O密集型的应用，例如UUID

用UUID作为主键的坏处：

- 写入的目标也可能已经疏导磁盘并从缓存中移除，或者还没有被加载到缓存中。InnoDB不得不先找到并且从磁盘读取目标页到内存，这导致了大量的随机I/O
- 因为写入是乱序的，InnoDB不得不频繁地做页分裂操作，会导致移动大量数据，一次插入最少需要修改三个页而不是一个页
- 由于频繁的页分裂，页会变的稀疏并被不规则填充，所以最终数据会有碎片
- UUID较长，不利于作为索引

`字符类型作为主键的通用替代方案`：[美团分布式ID生成项目](https://tech.meituan.com/2019/03/07/open-source-project-leaf.html)， 雪花ID

### 覆盖索引

如果一个索引包含（或者说覆盖）所有需要查询的字段的值，我们就称之为：覆盖索引

覆盖索引的好处：

- 索引条目远小于数据行大小，能够极大地提高性能，所以如果只需要读取索引，那么MySQL就会极大地减少数据访问量
- 因为索引是按照值顺序存储的，所以对于I/O密集型的范围查询会比随机从磁盘中读取每一行数据的I/O要少的多

如果不覆盖索引，则会产生`回表查询`， 先定位主键值，再定位行记录，它的性能较扫一遍索引树更低

### 使用索引扫描来做排序

MySQL有两种方式可以生成有序的结果：通过排序操作；或者按索引顺序扫描

如果EXPLAIN出来的type列的值为index，则说明使用了索引扫描排序

扫描索引本身是很快的，因为只需要从一条索引记录移动到紧接着的下一条记录。但如果索引无法覆盖所有的列，那就不得不扫描

一条索引记录就回表查询一次对应的行，这基本上属于随机I/O，因此按索引顺序读取数据的速度通常比顺序地全表扫描要慢

### 压缩（前缀压缩）索引

MyISAM使用前缀压缩来减少索引的大小，从而让更多的索引可以放入内存中，这在某种情况下可以极大地提高性能。默认只压缩

字符串，但通过参数设置也可以对整数压缩 ， MyISAM存储引擎不过多深究

### **冗余和重复索引**

**重复索引**：

- MySQL允许在相同列上创建多个索引，但这样需要单独维护重复的索引，并且优化查询的时候也需要逐个进行考虑，会影响性能，应该避免这么做

**冗余索引**

- 如果已经创建了索引（A, B），在创建索引（A），那么就是冗余索引，因为它只是前一个索引的前缀
- 冗余索引通常发生在表添加新索引的时候。如增加一个新的索引(A, B)，而没有扩展已有索引(A)，导致(A)成为冗余索引。或者将索引扩展为(A, 主键ID)，对InnoDB来说，主键已经包含在二级索引中了，因此也是冗余的

### 索引和锁

索引可以让查询锁定更少的行，如果你的查询从不访问那些不需要的行，那么就会锁定更少的行，从两个方面来看这对性能都有好

处：

- 虽然InnoDB的行锁的效率很高，内存使用也很少，但是锁定行的时候依然会带来额外开销
- 锁定需要的行会增加所争用并减少并发性

InnoDB只有在访问行的时候才会对其加锁。而索引能够减少InnoDB访问的行数，从而减少锁的数量。但这只有当InnoDB在存储引

擎能够过滤掉不需要的行时才有效，如果索引无法过滤掉无效的行，那么在InnoDB检索到数据并返回给服务器层之后，MySQL服务

器才能应用Where子句，这时已经无法避免锁定行了：InnoDB已经锁住了这些行，到适当的时候才释放。

Explain的Extra列显示“Using Where”，说明MySQL服务器将存储引擎返回行之后再应用where过滤条件。此时where子句以前的数据全都被加锁

InnoDB在二级索引上使用共享锁（读锁），但访问主键索引需要排它锁（写锁）

**InnoDB的行锁是建立在索引的基础之上的**，行锁锁的是索引，不是数据，所以提高并发写的能力要在查询字段添加索引

## 维护索引和表

使用正确的类型创建了表并加上了合适的索引后，还需要维护表和索引来确保它们正常工作，目的如下：

- 找到并修复损坏的表
- 维护准确的索引统计信息
- 减少碎片

### 找到并修复损坏的表

可以通过CHECK TABLE检查是否发生了表错误

可以用REPAIR TABLE或者一个不作任何操作的ALTER操作来修复表

### **更新索引统计信息**

- records_in_range()，通过向存储引擎传入两个边界值获取在这个范围大概有多少条记录
- info()，返回各种类型的数据，包括索引的基数（每个键值有多少条记录）

### 减少索引和数据的碎片

B-Tree索引可能导致碎片化，会导致查询效率降低。有三类数据碎片

- 行碎片：数据行被存储在多个片段中
- 行间碎片：逻辑上顺序的页，或者行在磁盘上不是顺序存储的
- 剩余空间碎片化：数据也中有大量的空余空间

对于MyISAM表，三类碎片都可能发生，InnoDB不会出现短小的碎片行，会移动短小的行并重写到一个片段中

## 总结

选择索引以及利用索引查询时的三个原则：

- 单行访问是很慢的。最好读取的块中包含尽可能多需要的行，使用索引可以创建位置引用以提升效率
- 按顺序访问范围数据是很快的，原因如下：
  1. 顺序I/O不需要多次磁盘寻道，比随机I/O快
  2. 如果服务器能够按顺序读取数据，那么就不再需要额外的排序操作，并且GROUP BY查询也无须再做排序和将行按组进行聚合计算了
- 索引覆盖查询是很快的

现实使用中，很难做到每一个查询都有完美的索引，这时候需要根据需求有所取舍地创建合适的索引，而非根据惯例一刀切

# 第6章 查询性能优化

对于高性能数据库操作，只靠设计最优的库表结构、建立最好的索引是不够的，还需要合理的设计查询。如果查询写得很糟糕，即使库表结构再合理、索引再合适，也无法实现高性能。查询优化、索引优化、库表结构优化需要齐头并进，一个不落。

## 为什么查询速度会慢

通常来说，查询的生命周期大致可以按照顺序来看：从客户端>>服务器>>在服务器上进行解析>>生成执行计划>>执行>>返回结果给客户端。其中执行可以认为是整个生命周期中最重要的阶段，这其中包括了大量为了检索数据到存储引擎的调用以及调用后的数据处理，包括排序、分组等。了解查询的生命周期、清楚查询的时间消耗情况对于优化查询有很大的意义。

## 慢查询基础：优化数据访问

查询性能低下的最基本的原因是访问的数据太多。大部分性能低下的查询都可以通过减少访问的数据量的方式进行优化。

1.确认应用程序是否在检索大量超过需要的数据。这通常意味着访问了太多的行，但有时候也可能是访问了太多的列。

2.确认MySQL服务器层是否在分析大量超过需要的数据行。



### 是否向数据库请求了不需要的数据

请求多余的数据会给MySQL服务器带来额外的负担，并增加网络开销，另外也会消耗应用服务器的CPU内存和资源。这里有一些典型案例：

1、查询不需要的记录：例如在新闻网站中取出100条记录，但是只是在页面上显示10条。实际上MySQL会查询出全部的结果，客户端的应用程序会接收全部的结果集数据，然后抛弃其中大部分数据。最简单有效的解决方法就是在这样的查询后面加上LIMIT。

2、多表关联时返回全部列，例如：

![image-20201020035237470](../images/image-20201020035237470.png)

3、总是取出全部的列：每次看到SELECT *的时候都需要怀疑是不是真的需要返回全部的列？取出全部列，会主优化器无法完成索引覆盖扫描这类优化，还会为服务器带来额外的IO、内存和CPU的消耗。如果应用程序使用了某种缓存机制，或者有其他考虑，获取超过需要的数据也可能有其好处，但不要忘记这样做的代价是什么。获取并缓存所有的列的查询，相比多个独立的只获取部分列的查询可能就更有好处。

4、重复查询相同的数据：不要不断地重复执行相同的查询，然后每次都返回完全相同的数据。当初次查询的时候将这个数据缓存起来，需要的时候从缓存中取出，这样性能显然更好。



### MySQL是否在扫描额外的记录

对于MySQL，最简单的衡量查询开销的三个指标有：响应时间、扫描的行数、返回的行数。这三个指标都会记录到MySQL的慢日志中，所以检查慢日志记录是找出扫描行数过多的查询的好办法。

**响应时间**

响应时间是两个部分之和：服务时间和排队时间，一般常见和重要的等待是IO和锁等待。

**扫描的行数和返回的行数**

分析查询时，查看该查询扫描的行数是非常有帮助的。一定程度上能够说明该查询找到需要的数据的效率高不高。理想的情况下扫描的行数和返回的行数应该是相同的。当然这只是理想情况。一般来说扫描的行数对返回的行数的比率通常很小，一般在1：1到10：1之间。

**扫描的行数和访问类型**

MySQL有好几种访问方式可以查找并返回一行结果。有些访问方式可能需要扫描很多行才能返回一行结果，也有些访问方式可能无须扫描就能返回结果。

在EXPLAIN语句的TYPE列返回了访问类型。如果查询没有办法找到合适的访问类型，那么解决的最好办法通常就是增加一个合适的索引。索引让MySQL以最高效、扫描行最少的方式找到需要的记录。

一般MySQL能够使用如下三种方式应用WHERE条件，从好到坏依次为：

1、在索引中使用WHERE条件来过滤不匹配的记录。这是在存储引擎层完成的。

2、使用索引覆盖扫描（在extra列中出现了using index）来返回记录，直接从索引中过滤不需要的记录并返回命中的结果。这是在MySQL服务器层完成的，但无须回表查询记录。

3、从数据表中返回数据，然后过滤不满足条件的记录（在extra列中出现using where）。这在MySQL服务器层完成，MySQL需要先从数据表读出记录然后过滤。

## 重构查询方式

###  一个复杂查询还是多个简单查询

MySQL内部每秒能够扫描内存中上百万行数据，相比之下，MySQL响应数据给客户端就慢得多了。在其他条件都相同的时候，使用尽可能少的查询当然是更好的。但是有时候，将一个大查询分解为多个小查询也是很有必要的。

### **切分查询**

有时候对于一个大查询我们需要“分而治之”，对于删除旧数据，如果用一个大的语句一次性完成的话，则可能需要一次性锁住很多数据、占满整个事务日志、耗尽系统资源、阻塞很多小的但重要的查询。将一个大的DELETE语句切分成多个较小的查询可以尽可能小地影响MySQL性能，同时还可以减少MySQL复制的延迟。例如我们需要每个月运行一次下面的查询：

![image-20201020035337049](../images/image-20201020035337049.png)

那么可以用类似下面的办法来完成同样的工作：

![image-20201020035459544](../images/image-20201020035459544.png)

### **分解关联查询**

![image-20201020035521674](../images/image-20201020035521674.png)

乍一看这样做并没有什么好处，但其有如下优势：

1、让缓存的效率更高。对MySQL的查询缓存来说，如果关联中的某个表发生了变化 ，那么就无法使用查询缓存了，而拆分后，如果某个表很少改变，那么该表的查询缓存能重复利用 。

2、将查询后，执行单个查询可以减少锁的竞争。

3、查询性能也有所提升，使用IN（）代替关联查询，可以让MySQL按照ID顺序进行查询，这比随机的关联要更高效。

## 查询执行基础

希望MySQL能够能更高的性能运行查询时，最好的办法就是弄清楚MySQL是如何优化和执行查询的。

<img src="../images/image-20201020035549242.png" alt="image-20201020035549242"  />

### **MySQL客户端/服务端通信协议**

MySQL客户端和服务器之间的通信协议是“半双工”的，在任何一个时刻，要么由服务器向客户端向服务端发送数据，要么是由客户端向服务器发送数据，这两个动作不能同时发生。

一旦客户端发送了请求，它能做的事情就只是等待结果了，如果查询太大，服务端会拒绝接收更多的数据并抛出相应错误，所以参数max_allowed_packet就特别重要。相反，一般服务器响应给用户的数据通常很多，由多个数据包组成。当服务器开始响应客户端请求时，客户端必须完整地接收整个返回结果，而不能简单地只取前面几条结果，然后主服务器停止发送数据。这种情况下，客户端若接收完整的结果，然后取前面几条需要的结果，或者接收完几条结果然后粗暴地断开连接，都不是好主意。这也是必要的时候需要在查询中加上limit限制的原因。

换一种方式解释这种行为：当客户端从服务器取数据时，看起来是一个拉数据的过程，但实际上是MySQL在向客户端推数据的过程。客户端不断地接收从服务器推送的数据，客户端也没法让服务器停下来。

当使用多数连接MySQL的库函数从MySQL获取数据时，其结果看起来都像是从MySQL服务器获取数据，而实际上都是从这个库函数的缓存获取数据。多数情况下这没什么问题，但是如果需要返回一个很大的结果集时，这样做并不好，因为库函数会花很多时间和内存来存储所有的结果集。如果能尽早开始处理这些数据，就能大大减少内在的消耗，这种情况下可以不使用缓存来记录结果而是直接处理。PHP的 mysql_query()，此时数据已经到了PHP的缓存中，而mysql_unbuffered_query()不会缓存结果。

查询状态：可以使用SHOW FULL PROCESSLIST命令查看查询的执行状态。Sleep、Query、Locked、Analyzing and statistics、Copying to tmp table[on disk]、Sorting result、Sending data

###  **查询缓存**

在解析一个查询语句之前，如果查询缓存是打开的，那么MySQL会优先检查这个查询是否命中查询缓存中的数据。这是检查是通过一个对大小写敏感的哈希查找实现的。如果当前的查询恰好命中了查询缓存，那么在返回查询结果之前MySQL会检查一次用户权限。如果权限没有问题，MySQL会跳过执行阶段，直接从缓存中拿到结果并返回给客户端。

###  **查询优化处理**

查询生命周期的下一步是将一个SQL转换成一个执行计划，MySQL再依照这个执行计划和存储引擎进行交互。这包括多个子阶段：解析SQL、预处理、优化SQL执行计划。

1、语法解析器和预处理首先MySQL通过关键字将SQL语句进行解析，并生成一棵解析树。MySQL解析器将使用MySQL语法规则验证和解析查询。例如是否使用错误的关键字，或者使用关键字的顺序是否正确，引号是否能前后正确匹配等。

2、预处理器则根据一些MySQL规则进一步检查解析树是否合法，例如检查数据表和数据列是否存在，还会解析名字和别名看它们是否有歧义。

3、一下步预处理会验证权限。



**查询优化器**：一条语句 可以有很多种执行方式，最后都返回相同的结果。优化器的作用就是找到最好的执行计划。MySQL使用基于成本的优化器，它将尝试预测一个查询使用某种执行计划时的成本，并选择其中成本最小的一个。成本的最小单位是随机读取一个4K的数据页的成本，并加入一些因子来估算某引动操作的代价。可以通过查询当前会话的Last_query_cost的值来得知MySQL计算的当前查询的成本。

这是根据一系列的统计信息计算得来的：每个表或者索引的页面个数、索引的基数（索引中不同值的数量）、索引和数据行的长度、索引分布情况。

当然很多原因会导致MySQL优化器选择错误的执行计划：例如统计信息不准确或执行计划中的成本估算不等同于实际执行的成本。

**MySQL如何执行关联查询**：MySQL对任何关联都执行嵌套循环关联操作，即MySQL先在一个表中循环取出单条数据，然后再嵌套循环到一个表中寻找匹配的行，依次下去直到找到的有匹配的行为止。然后根据各个表匹配的行，返回查询中需要的各个列。**（嵌套循环关联）**

![image-20201020035732618](../images/image-20201020035732618.png)

**执行计划**：MySQL生成查询的一棵指令树，然后通过存储引擎执行完成这棵指令树并返回结果。最终的执行计划包含了重构查询的全部信息。如果对某个查询执行EXPLAIN EXTENDED，再执行SHOW WARNINGS，就可以看到重构出的查询。

MySQL的执行计划是一棵左侧深度优先的树。

![image-20201020035748073](../images/image-20201020035748073.png)

不过，如果有超过n个表的关联，那么需要检查n的阶乘种关联顺序。我们称之为所有可能的执行计划的“搜索空间”。实际上，当需要关联的表超过optimizer_search_depth的限制的时候，就会选择“贪婪”搜索模式。



**排序优化**：无论如何排序都是一个成本很高的操作，所以从性能角度考虑，应尽可能避免排序或者尽可能避免对大量数据进行排序。如果需要排序的数据量小于排序缓冲区，MySQL使用内存进行“快速排序”操作。如果内存不够排序，那么MySQL会先将数据分块，对每个独立的块使用“快速排序”进行排序，并将各个块的排序结果存放在磁盘上，然后将各个排序的块进行合并，最手返回排序结果。

MySQL有两种排序方法：

  两次传输排序（旧版），读取行指针和需要排序的字段，对其进行排序，然后再根据排序结果读取所需要的数据行。显然是两次传输，特别是读取排序后的数据时（第二次）大量随机I/O，所以两次传输成本高。

  单次传输排序（新版），一次读取出所有需要的或SQL查询指定的列，然后根据排序列，排序，直接返回排序后的结果。顺序I/O，缺点：如果列多，额外占用空间。

MySQL在进行文件排序时需要使用的临时存储空间可能会比想象的要大得多，因为MySQL在排序时，对每一个排序记录都会分配一个足够长的定长空间来存放。这个定长空间必须足够以容纳其中最长的字符串。

在关联查询的时候如果需要排序，MySQL会分两种情况来处理这样的文件排序。如果ORDER BY子句的所有列都来自关联的第一个表，那么MySQL在关联处理第一个表时就进行文件排序。如果是这样那么在MySQL的EXPLAIN结果中可以看到Extra字段会有Using filesort。除此之外的所有情况，MySQL都会将关联的结果存放在一个临时表中，然后在所有的关联都结束后，再进行文件排序。这种情况下Extra字段可以看到Using temporary;Using filesort。如果查询中有LIMIT的话，LIMIT也会在排序之后应用，所以即使需要返回较少的数据，临时表和需要排序的数据量仍然会非常大。

### **查询执行引擎**

相对于查询优化，查询执行简单些了，MySQL只根据执行计划输出的指令逐步执行。指令都是调用存储引擎的API来完成，一般称为 handler API，实际上，MySQL优化阶段为每个表都创建了一个 handler 实例，用 handler 实例获取表的相关信息（列名、索引统计信息等）。

存储引擎接口有着非常丰富的功能，但是底层接口却只有几十个，这些接口像搭积木一样能够完成查询的大部分操作。例如，有一个查询某个索引的第一行的接口，再有一个查询某个索引条件的下一条目的功能，有了这两个功能就可以完成全索引扫描操作。

### **返回结果给客户端**

查询执行的最后一个阶段就是将结果返回给客户端。即使查询不需要返回结果集给客户端，MySQL仍然会返回这个查询的一些信息，例如该查询影响到的行数。

MySQL将结果集返回客户端是一个增量、逐步返回的过程。一旦服务器处理完最后一个关联表，开始生成第一条结果时，MySQL就可以开始向客户端逐步返回结果集了。

这样处理有两个好处：服务端无须存储太多的结果，也就不会因为要返回太多结果而消耗太多内存。另外，这样的处理也让MySQL客户端第一时间获得返回的结果。



当然，优化器存在其局限性，以及某些特定的优化类型，有兴趣的可以在书中找到答案。

## MySQL查询执行器的局限性

1. 子查询相对糟糕(不是绝对),如子查询用in
2. 联表查询与子查询根据场景不同有不同优势
3. MySQL无法将限制条件下推到子查询
4. 索引合并优化
5. MySQL无法利用多核特性并行执行查询
6. MySQL并不支持哈希关联, MariaDB已经实现了真正的哈希关联
7. 松散索引扫描,无法按照不连续的方式扫描一个索引
8. 最大值最小值函数的优化一般
9. 不允许同一张表上同时查询和更新, 如update set 等于 select 自己.解决方法,可以通过关联临时表

## 查询执行器的提示（hint）

1. 设置查询优化器参数,可以阅读官方手册
2. 一般除非需要,修改查询优化器参数会提高维护成本

## 优化特定类型查询

1. 关联查询: on的列加索引; 使用group by和order by 只使用一个表的列可以利用索引
2. 优化LIMIT分页: 尽量使用覆盖索引
3. 子查询: 尽量使用关联查询替换
4. 静态查询分析: Percona Toolkit中的pt-query-advisor能解析查询日志,分析查询模式
5. 使用用户自定义变量: 无法使用查询缓存,可能被优化器优化掉

# 第7章 MySQL高级特性

## 分区表

###  应用

1. 表非常大无法全部放在内存中,或者只在表的最后部分有热点数据其他均是历史数据
2. 分区表的数据更容易维护
3. 分区表的数据可以分布在不同的物理设备上
4. 使用分区表避免某些瓶颈,如InnoDB单个索引的互斥访问
5. 备份和恢复独立分区,对于大数据集效果较好

### 限制

1. 一个表最多1024个分区
2. 分区表达式必须是整数或返回整数的表达式
3. 如果分区字段有主键或唯一索引列,那么所有主键列和唯一索引都必须包含进来
4. 分区表中无法使用外键约束

### 原理

1. 分区表由多个相关的底层表实现,存储引擎管理它们跟管理普通表一样
2. select 查询: 分区层打开并锁住所有底层表,优化器判断是否过滤分区,在调用存储引擎api访问各个分区数据
3. insert: 分区层打开并锁住所有底层表,确定分区,写入
4. delete: 分区层打开并锁住所有底层表,确定数据所在分区,删除
5. update: 分区层打开并锁住所有底层表,确定分区,取出数据,更新,确定分区,写入
6. 打开并锁住所有底层表: 如果存储引擎实现行级锁如InnoDB,则会在分区层释放表锁

###  分区表类型

1. 根据范围进行分区: 每个分区储存落在某个范围的记录
2. 根据键值进行分区,减少InnoDB互斥量竞争
3. 使用数学模函数进行分区,然后将数据轮询放入不同的分区,适用于只想保留几天的数据

### 使用

1. 问题回顾:数据量很大时,除非是索引覆盖查询,否则数据库需要根据索引扫描回表查询,产生大量的随机IO,数据库响应时间很大
2. 全量扫描数据不要索引,根据分区定位数据位置
3. 索引数据,分离热点. 将热点数据单独放在一个分区
4. NULL值会使分区过滤无效: 分区表达式接收NULL值并将其放到第一个分区导致查询时多查分区.解决方法:创建第一个无用分区存放NULL值数据
5. 分区列和索引列不匹配,查询无法进行分区过滤
6. 选择分区成本高,插入大量数据时都需要扫描分区定义找到分区
7. 打开并锁住所有底层表的成本可能很高
8. 维护分区的成本很高,同alter一样创建临时表然后拷贝数据
9. 所有分区都必须使用相同的存储引擎

###  查询优化

1. 在where条件带入分区列
2. 创建分区时可以使用表达式,但是查询时只能在使用分区列本身进行比较时才能过滤分区,而不能根据表达式的值过滤分区

##  视图

视图本身是一个虚拟表,不存放任何数据,不能对视图创建触发器

###  算法

1. 合并算法: 将存放的视图sql和用户发起的查询sql合并后执行
2. 临时表算法: 由存放的视图sql先创建临时表后根据用户的查询sql查询返回

###  可更新视图

1. 可以通过更新视图更新相关表, 所有临时表算法实现的视图都无法更新

###  视图对性能的影响

1. 一般情况视图不能提升性能,在某些情况下可以帮助提升性能,需要做比较详细的测试
2. 视图还可以实现基于列的权限控制不用真正创建列权限

###  视图的限制

1. 不保存视图定义的原始sql语句
2. 查看视图创建的语句,可以通过使用视图的.frm文件的最后一行获取一些信息

##  外键约束

1. InnoDB强制外键使用索引
2. 查询需要额外访问一些表,需要额外的锁容易导致一些死锁
3. 如果使用外键做约束,通常在应用程序里实现会更好

##  内部存储代码

### 优点

1. 离数据最近,节省带宽和网络延迟
2. 帮助提升安全性,应用程序可以通过存储过程访问那些没有权限的表
3. 服务器端可以缓存存储过程的执行计划
4. 维护方便,便于分工

###  缺点

1. 调试困难,难以定位问题
2. 存储代码效率相对差
3. 增加维护复杂性,存储过程会给数据库服务器增加额外压力
4. 存在安全隐患,没有什么选项可以控制存储程序的资源消耗,所以一个小错误可能直接把服务器拖死

###  存储过程和函数

1. 优化器无法评估存储函数的执行成本
2. 每个连接都有独立的存储过程的执行计划缓存,多个连接调用同一个存储过程会浪费缓存空间反复缓存同样的执行计划

### 触发器

1. 每个表的每个事件只能一个
2. MySQL只支持基于行的触发,如果变更的数据集非常庞大的化效率会很低
3. 触发器的问题很难排查
4. 可能导致死锁和锁等待
5. 实现一些约束,系统维护任务及更新反范式化数据的时候会比较有用

###  事件

1. 类似Linux的定时任务

##  游标

1. MySQL在服务器端提供只读的,单向的游标
2. 一个存储过程中可以有多个游标,也可以嵌套

##  绑定变量

1. 创建一个绑定变量sql时客户端向服务器发送了一个sql语句原型
2. 服务器端解析并存储这个sql语句的部分执行计划返回客户端一个sql语句处理句柄
3. 可以使用问号作为sql的占位,在使用sql接口执行时赋予变量值

## 插件

1. 存储过程插件
2. 后台插件: 如Percona Server中包含的Handler-Socket
3. INFORMATION_SCHEMA插件
4. 全文解析插件: 可以对文档进行分词处理
5. 审计插件: 可以用作记录事件日志
6. 认证插件: 扩展认证功能

##  字符集和校对

1. 字符集是指一种从二进制编码到某类字符符号的映射
2. 校对是指一组用于某个字符集的排序规则

###  创建对象时的默认设置

1. 服务器,数据库,表都有默认的字符集和校对规则,这是一个逐层继承的默认设置
2. 创建数据库时根据character_set_server设置来设定默认字符集

###  服务器和客户端通信设置

1. 服务端总是假设客户端按照character_set_client设置的字符来传输数据和sql语句
2. 服务器端收到sql语句后根据character_set_connection转换成字符串
3. 服务器端返回数据时会将其转换为character_set_result

###  选择字符集和校对规则

1. 极简原则: 先为服务器选择合理的字符集在根据实际情况让某些列选择合适的字符集

###  对查询的影响

1. 不同字符集和校对规则之间的转换会带来额外的开销
2. 排序查询要求的字符集与服务器数据的字符集相同时才能利用索引进行排序
3. 当两个字符集不同列关联两个表时,MySQL会尝试转换其中一个列的字符集

## 全文索引

1. 自然语言的全文索引: 相关度是基于匹配的关键词个数及关键词在文档中出现的次数,整个索引中出现次数越少的词语匹配的相关度越高
2. 布尔全文索引: 只有MyISAM才能使用
3. 平时没接触过,有兴趣者请自行google

##  分布式XA事务

1. 事务协调器保证所有事务参与者完成工作,通知所有事务提交
2. 内部XA事务: 存储引擎提交的同时,需要将提交的信息写入二进制日志
3. 外部XA事务: XA事务是一种在多个服务器之间同步数据的方法,如果由于不能使用MySQL本身的复制或者性能并不是瓶颈可以尝试使用

## 查询缓存

1. 查询缓存系统会跟踪查询中涉及的每个表,如果表发生变化则缓存数据失效
2. 缓存存放在一个引用表中,通过一个哈希值引用,哈希值包括查询本身,查询数据库等信息
3. 当sql语句和客户端发送过来的其他原始信息,任何字符上的不同都会导致缓存不命中
4. 打开查询缓存对读和写都会带来额外的消耗
5. InnoDB事务修改表时,会将这个表对应的查询缓存都设置失效
6. 查询缓存被发现是一个影响服务器扩展性的因素
7. 如果缓存了大量的查询结果,那么失效操作可能会造成系统僵死.因为靠一个全局锁保护,所有该操作都要等锁
8. 减少碎片, 选择合适的query_cache_min_res_unit可以减少内存浪费
9. 对于写密集型的应用,直接禁用更好
10. 高并发环境也不适合.只有明确缓存的好处才使用
11. 查询缓存的替代方案: 客户端缓存

# 第8章 优化服务器设置

1. MySQL有大量的可以修改的参数,但不应该随便修改.应该将更多时间花在schema的优化,索引,查询设计上
2. 配置文件路径: 通常在/etc/my.cnf
3. 不建议动态修改变量,因为可能导致意外的副作用
4. 通过基准测试迭代优化
5. 具体配置项设置请参照官网手册,这里只提及部分

## 配置内存使用

1. 确定可使用内存上限
2. 每个连接使用多少内存,如排序缓冲和临时表
3. 确定操作系统内存使用量
4. 把剩下的分配给缓存,如InnoDB缓存池

##  配置MySQL的I/O行为

1. 有些配置项影响如何同步数据到磁盘及如何恢复操作,这对性能影响很大,而且表现了性能和数据安全之间的平衡

### InnoDB I/O配置

1. 重要配置: InnoDB日志文件大小,InnoDB怎样刷新日志缓冲,InnoDB怎样执行I/O
2. InnoDB使用日志减少提交事务时开销,不用每个事务提交时把缓冲池的脏块刷到磁盘中
3. 事务日志可以把随机IO变成顺序IO,同时如果发生断电,InnoDB可以重放日志恢复已经提交的事务
4. sync_binlog选项控制MySQL怎么刷新二进制日志到磁盘
5. 把二进制日志放到一个带有电池保护的写缓存的RAID卷可以极大的提升性能

### MyISAM的I/O配置

1. 因为MyISAM表每次写入都会将索引变更刷新到磁盘
2. 批量操作时,通过设置delay_key_write可以延迟索引写入,可以提升性能
3. 配置MyISAM怎样尝试从损坏中恢复

##  配置MySQL并发

###  InnoDB并发配置

1. 如果在InnoDB并发方面有问题,解决方案通常是升级服务器
2. innodb_thread_concurrency: 限制一次性可以有多少线程进入内核(根据实践取合适值)
3. innodb_thread_sleep_delay: 线程第一次进入内核失败等的时间,如果还不能进入则放入等待线程队列
4. innodb_commit_concurrency: 控制有多少线程可以在同一时间提交
5. 使用线程池限制并发: MariaDB已经实现

###  MyISAM并发配置

1. concurrency_insert: 配置MyISAM打开并发插入

##  其他

1. 基于工作负载的配置: 利用工具分析并调整配置
2. max_connections: 保证服务器不会因应用程序激增的连接而不堪重负
3. 安全和稳定的设置: 感兴趣者请自行google
4. 高级InnoDB设置: 感兴趣者请自行google
5. InnoDB两个重要配置: innodb_buffer_pool_size和innodb_log_file_size

[《高性能MySQL》读书笔记－－优化服务器设置](https://blog.csdn.net/xifeijian/article/details/45702235)

[影响MySQL性能的五大配置参数](https://blog.csdn.net/xifeijian/article/details/19775017)

# 第9章 操作系统和硬件优化

（详略）

# 第10章 复制

MySQL内建的复制功能是构建基于MySQL的大规模,高性能应用的基础.同时也是高可用性,可扩展性,灾难恢复,备份及数据仓库等工作的基础

##  概述

1. 解决问题: 让一台服务器的数据与其他服务器保持同步.主库可以同步到多台备库,备库本身也可以配置为另一台服务器的主库
2. 复制原理: 通过在主库上记录二进制日志,在备库重放日志的方式实现异步的数据复制
3. 复制方式: 基于行的复制和基于语句的复制
4. 向后兼容: 新版本只能作为老版本的备库,反之不行

##  用途

1. 数据分布: 在不同地理位置分布数据备份,可以随意停止或开始复制.基于行比基于语句带宽压力更大
2. 负载均衡: 将读操作分布到多个服务器上
3. 备份: 复制是备份的一项有意义的技术补充
4. 高可用性和故障切换: 避免单点失败
5. MySQL升级测试: 一种普遍做法是使用一个更高版本的MySQL作为备库保证实例升级前查询能够在备库按照预期执行

##  过程

1. 主库把数据更改记录到二进制日志(Binary Log)
2. 备库将主库上的日志复制到自己的中继日志(Relay Log)
3. 备库读取中继日志中的事件,将其重放到备库数据上
4. 局限: 主库上并发运行的查询在备库只能串行化执行,因为只有一个sql线程重放中继日志事件,这是很多工作负载的性能瓶颈

## 复制配置

1. 在每台服务器上创建复制账号: 需要REPLICATION SLAVE权限
2. 配置主库和备库: 每个服务器的ID需要唯一不能冲突
3. 通知备库连接到主库并从主库复制数据
4. CHANGE MASTER TO: 指定备库连接的主库设置
5. SHOW SLAVE STATUS: 检查复制是否正确执行
6. START SLAVE: 开始复制
7. SHOW PROCESSLIST: 查看复制线程,IO线程(发送或获取日志),SQL线程(重放日志)
8. 推荐配置: 开启sync_binlog

##  从另一个服务器开始复制

问题: 主库已经运行一段时间,用一台新安装的备库与之同步
保持同步条件:

1. 某个时间点的主库的数据快照
2. 主库当前的二进制日志文件,和获得数据快照时在该二进制日志文件中的偏移量.通过这两个可以确定二进制日志的位置
3. 从快照时间到现在的二进制日志

克隆备库方法:

1. 冷备份: 关闭主库,复制数据.主库重启后会使用新的二进制文件,在备库指向这个文件的起始处
2. 热备份:如果只有MyISAM,可以通过mysqlhotcopy或rsync来复制数据
3. 如果只包含InnoDB: 可以使用mysqldump转储主库数据并加载到备库,然后设置相应的二进制日志坐标
4. 使用快照或备份: 使用主库的快照或者备份初始化备库,然后指定二进制日志坐标
5. 使用Percona Xtrabackup: 备份时不阻塞服务器操作,可以在不影响主库情况下设置备库
6. 使用另外的备库: 实质就是把另外的备库当成主库进行数据克隆

##  复制的原理

###  基于语句的复制

1. 主库会记录那些造成数据更改的查询
2. MySQL5.0之前只支持基于语句的复制
3. 对于函数,存储过程和触发器在基于语句的复制模式可能存在问题
4. 更新必须是串行,需要更多的锁

###  基于行的复制

1. 将实际的数据记录在二进制日志
2. 能够更高效复制数据
3. 基于行的复制事件格式,对人不可读,可以使用mysqlbinlog
4. 很难进行时间点恢复
5. 有些操作,如全表更新(update)复制开销会很大

##  复制拓扑

### 基本原则

1. 一个MySQL备库实例只能有一个主库
2. 每个备库必须有一个唯一的服务器id
3. 一个主库可以有多个备库
4. 如果打开log_slave_update一个备库可以把其主库上的数据变化传播到其他备库

### 一主多备

1. 适用于少量写和大量读,可以把读分摊到多个备库上
2. 当作待用的主库
3. 放到远程数据中心,用作灾难恢复
4. 作为备份,培训,开发或测试服务器

### 双主复制

1. 个数据库互为主库和备库
2. 容易造成数据不同步
3. 通常并不建议使用这种模式

### 主动被动的双主模式

1. 类似双主复制,把其中一台配置为只读
2. 类似于创建一个热备份
3. 可以用作执行读操作,备份,离线维护及升级

###  有备库的双主模式

1. 双主模式下,各自有备库

### 主库,分发主库和备库

1. 问题: 备库足够多时会对主库造成很大的负载
2. 方案: 将其中部分备库当成主库,分发给更多的备库
3. 通过分发主库,可以对二进制日志事件执行过滤和重写规则

### 复制管理和维护

1. 监控复制: SHOW MASTER STATUS查看主库状态, SHOW BINLOG EVENTS查看复制事件
2. 测量备库延迟: 可以使用Percona Toolkit里的pt-hearbeat
3. 确定主备是否一致
4. 备库换主库: 难点在于获取新主库合适的二进制日志位置
5. 备库提升为主库分为计划内提升和计划外提升

### 计划内提升

1. 停止向老的主库写入
2. 备库赶上主库
3. 备库设置为主库
4. 将备库和写操作指向新主库,然后开启主库的写入

###  计划外提升

当主库崩溃时,需要提升一台备库替代

1. 确定最新的备库
2. 让所有备库执行完从崩溃前主库获得的中继日志,如果未完成则更换主库,会丢失原先的日志事件
3. 重新完成主备的配置

##  复制的问题和解决方案

### 数据损坏或丢失

1. 主库意外关闭: 主库开启sync_binlog避免事件丢失,使用Percona Toolkit中的pt-table-checksum检查主备一致性
2. 备库意外关闭: 重启后观察MySQL错误日志,想方法获取备库指向主库的日志偏移量
3. 主库上的二进制日志损坏: 跳过所有损坏的事件,手动找到一个完好的事件开始
4. 备库上的中继日志损坏: MySQL5.5后能在崩溃后自动重新获取中继日志
5. 二进制日志于InnoDB事务日志不同步: 除非备库中继日志有保存,否则自求多福

###  其他

1. 如果使用myisam,在关闭Mysql前需要确保已经运行了stop slave,否则在服务器关闭时会kill所有正在运行的查询.
2. 如果是事务型,失败的更新会在主库上回滚而且不会记录到二进制日志
3. 避免混用事务和非事务: 如果备库发生死锁而主库没有,事务型会回滚而非事务型则不会造成不同步
4. 主库和备库使用不同存储引擎容易导致问题
5. 不唯一和未定义备库服务器id
6. 避免在主库上创建备库上没有的表,因为复制可能中断
7. 基于语句复制时,主库上没有安全使用临时表的方法.丢失临时表: 备库崩溃时,任何复制线程拥有的临时表都会丢失,重启备库后所有依赖临时表的语句都会失败
8. InnoDB加锁读引起的锁争用: 将大命令拆成小命令可以有效减少锁竞争
9. 过大的复制延迟: 定位执行慢的语句,改善机器配置
10. 其他: 查看官网手册

## 复制高级特性

1. 半同步复制: 当提交事务,客户端收到查询结束反馈前必须保证二进制日志已经传输到至少一台备库上,主库将事务提交到磁盘上之后会增加一些延迟
2. 复制心跳: 保证备库一直与主库相联系,如果出现断开的网络连接,备库会注意到丢失的心跳数据

## 其他复制技术

1. Percona XtraDB Cluster的同步复制
2. Tungsten

# 第11章 可扩展的MySQL

可扩展性: 通过增加资源提升容量的能力

## 考虑负载

容量可以简单地认为是处理负载的能力,考虑负载可从以下几个角度

1. 数据量: 很多应用从不物理删除任何数据,应用所积累的数据量是可扩展的普遍挑战
2. 用户量: 更多的用户意味着更多的事务,更多的复杂查询
3. 用户活跃度
4. 相关数据集的大小

## 规划可扩展性

1. 估算需要承担的负载到底有多少
2. 大致正确地估计日程表
3. 应用的功能完成多少
4. 预期的最大负载是多少
5. 如果依赖系统的每个部分分担负载,某个部分失效时会发生什么

##  向上扩展(垂直扩展)

1. 单台服务器增加各种高性能硬件
2. 烧钱有效的方法
3. 不应该无限制向上扩展

## 向外扩展

1. 策略: 复制,拆分,数据分片
2. 按功能拆分: 常见做法,根据功能将应用部署在不同服务器,并使用专用的数据库服务器

###  数据分片

数据分片是目前扩展大型MySQL最通用且最成功的方法

1. 应用设计初期考虑到,后期实现就比较容易,否则很难将应用从单一数据存储转换为分片架构
2. 文中举例: 通过用户id来对文章和评论进行分片,而将用户的信息保留在单个节点上
3. 数据库访问抽象层,降低应用和分片数据之间通信的复杂度
4. 如非必要尽量不分片
5. 数据分片最大的挑战就是查找和获取数据
6. 类似于表分区,选择分区键和数据分片方式是关键,具体请细查

## 通过集群扩展

1. 可以使用集群或数据库分布式技术根据场景适当解决一些问题
2. 书中提到技术: NDB Cluster, Clustrix等技术

## 向内扩展

1. 对不再需要的数据进行归档和清理
2. 需要考虑对应用的影响
3. 需要考虑数据逻辑的一致性,例如清理A表历史数据时需要考虑所有关联数据的处理
4. 冷热数据分离

## 负载均衡

###  目的

1. 可扩展性: 如读写分离时从备库读数据
2. 高效性: 把更多工作分配给更好的机器
3. 可用性: 使用时刻保持可用的服务器
4. 透明性: 客户端无需知道服务器
5. 一致性: 如果应用是有状态的,负载均衡器就应该将相关的查询指向同一个服务器

###  直接连接

#### 复制上的读写分离

1. 基于查询分离: 将不能容忍脏数据的查询分配到主库,其他分配到备库
2. 基于脏数据分离: 让应用检查复制延迟,许多报表类应用使用这个策略
3. 基于会话分离: 可以在会话层做一个标记,如果用户修改了数据,则一段时间内总是指向主库
4. 基于版本分离: 给用户的操作增加版本号,检查版本号决定从主库还是备库读取数据

####  修改DNS名

1. 通过变更DNS名指定的服务器实现
2. 缺点很多,不建议

####  转移IP地址

1. 在服务器之间转移虚拟地址
2. 给服务器分配固定的ip地址,为每个逻辑上的服务使用一个虚拟ip地址

### 引入中间件

1. 负载均衡器,如HAproxy
2. 负载均衡算法: 随机, 轮询,最少连接数,最快响应,哈希,权重
3. 服务器池中增加或移除服务器: 在配置连接池中的服务器时,要保证有足够多未使用的容量

# 第12章 高可用性

1. 高可用性意味着更少的宕机时间

## 宕机原因

1. 磁盘空间不足
2. 糟糕的sql或者服务器bug引起
3. 糟糕的表和索引设计
4. 复制问题通常由于主备数据不一致导致

##  高可用性实现

1. 衡量指标: 平均失效时间(MTBF), 平均恢复时间(MTTR)
2. 避免问题: 适当的配置,监控和规范
3. 保证在宕机时能快速恢复,系统制造冗余,具备故障转移能力

### 避免单点失效

1. 通过增加冗余避免
2. 共享存储或磁盘复制,如果服务器挂了,备用服务器可以挂载相同的文件系统执行需要的恢复操作
3. MySQL同步复制

### 故障转移和故障恢复

1. 提升备库或切换角色
2. 虚拟IP地址或IP接管: 当MySQL实例失效时可以将IP地址转移到另一台MySQL服务器上
3. 使用中间件解决方案

# 第13章 云端的MySQL

（略）

# 第14章 应用层优化

## Web服务器问题

Apache不适合用作通用Web服务器（既处理动态脚本也处理静态文件）：Apache对于静态文件的请求存在资源浪费，进程会复用，如果前一次处理的是动态语言脚本的请求，在请求结束后并不会释放所有的内存给操作系统，这样会造成一个占用内存很多的进程来为一个很小的请求服务的情况。同样的，这些被复用的进程也可能会保持大量MySQL连接，从而浪费MySQL资源。 总之，不要使用Apache来做静态内容服务，或者至少和动态服务使用不同的Apache实例。

使用缓存代理服务（Squid、Varnish），防止所有请求到达Web服务器。

打开gzip压缩。

不要为用于长距离连接的Apache配置启用Keep-Alive选项，因为这会使得重量级的Apache进程存活很长时间。

## 缓存

在实践中发现从Nginx的内存中获取内容比从缓存代理磁盘中获取数据要快。

（讲的太乱，也有一点过时，没有介绍redis，略）

# 第15章 备份与恢复

## 设计MySQL备份方案考虑点

1. 在线备份还是离线备份
2. 逻辑备份还是物理备份
3. 非显著数据: 如二进制日志和InnoDB事务日志
4. 代码: 存储过程,触发器
5. 服务器配置和复制配置
6. 外部配置,管理脚本
7. 增量备份和差异备份
8. 存储引擎和数据一致性

## 备份数据方式

1. 文件系统中或SAN快照中直接复制数据文件
2. Percona XtraBackup 做热备份

## InnoDB崩溃恢复

1. 二级索引损坏: 使用OPTIMIZE TABLE修复损坏的二级索引,此外可以通过构建一个新表重建受影响的索引
2. 聚簇索引损坏: innodb_force_recovery导出表
3. 损坏系统结构: 系统结构包括事务日志等,可能需要做整个数据库的导出和还原,因为InnoDB内部绝大部分的工作可能受影响

# 第16章 MySQL用户工具

工欲善其事,必先利其器

## 接口工具

1. MySQL Workbench: 一站式的工具
2. SQLyog: 可视化工具之一

## 命令行工具集

1. Percona Toolkit
2. MySQL Workbench 工具集

## SQL实用集

1. common_schema
2. MySQL Forge

## 监测工具

1. Nagios
2. Zabbix: 同时支持监控和指标收集的完整系统
3. Zenoss: Python写的
4. Hyperic HQ: 基于Java

## Innotop命令行监控

主要包括以下功能

1. 事务列表
2. 当前运行的查询
3. 当前锁和锁等待列表
4. 服务器状态和变量汇总信息
5. InnoDB内部信息
6. 复制监控

# 附录A MySQL分支与变种

（略）

# 附录B MySQL服务器状态

PERFORMANCE_SCHEMA库、INFORMATION_SCHEMA库、SHOW命令。 （详略）

# 附录C 大文件传输

（略）

# 附录D EXPLAIN

EXPLAIN命令用于查看查询优化器选择的查询计划。被标记了EXPLAIN的查询会返回关于执行计划中每一步的信息，而不是执行它。(实际上，如果查询在from子句中包括子查询，那么MySQL实际上会执行子查询) EXPLAIN只是一个近似结果。

## EXPLAIN的结果

`id` 标识SELECT所属的行，如果在语句中没有子查询或者联合查询，那么只会有一行（因为只有1个SELECT），且id值为1.

```
select_type
```

1. SIMPLE 意味着该查询不包含子查询和UNION，如果查询有任何复杂的子部分，则最外层部分标记为PRIMARY（id为1的查询）。
2. SUBQUERY 包含在SELECT列表中的子查询（即不是位于FROM子句中的查询）；
3. DERIVED 包含在FROM子句中的子查询；
4. UNION 在UNION查询中的第二个和随后的SELECT被标记为UNION；
5. UNION RESULT 用来从UNION的匿名临时表检索结果的SELECT；

`table` 表示正在访问哪个表（包括匿名临时表，比如derived1），可以在这一列从上往下观察MySQL的关联优化器为查询选择的关联顺序。

`type` 访问类型，决定如何查找表中的行，从最差到最优排列如下：

1. ALL 全表扫描；
2. index 按索引次序全表扫描，主要优点是避免了排序，缺点是当随机访问时开销非常大；如果在Extra列中有Using index，说明MySQL正在使用覆盖索引，即只扫描索引的数据。
3. range 有限制的索引扫描（带有between或where >等条件的查询）； 注意，当使用IN()、OR()来查询时，虽然也显示范围扫描，但是其实是相当不同的访问类型，在性能上有重要的差异。
4. ref 索引查找，返回所有匹配某个值的行，这个值可能是一个常数或者来自多表查询前一个表里的结果值；
5. eq_ref 也是索引查找，且MySQL知道最多只返回一条符合条件的记录（使用主键或者唯一性索引查询），MySQL对于这类访问优化的非常好；
6. const、system 当MySQL能对查询的某部分进行优化并将其转换为一个常量时，它就会使用这些访问类型；
7. NULL 意味着MySQL能在优化阶段分解查询语句，在执行阶段甚至用不着再访问表或者索引；

`possible_keys` 显示查询可以使用哪些索引。

`key` 显示MySQL决定采用哪个索引来优化对表的访问。

`key_len` 显示MySQL在索引里使用的字节数。

`ref` 显示之前的表在key列记录的索引中查找值所用的列或常量。

`rows` MySQL估计为了找到所需的行而要读取的行数。

`filtered` 在使用EXPLAIN EXTENED时才会出现，查询结果记录数占总记录数的百分比。

```
Extra
```

1. Using index：表示将使用覆盖索引；
2. Using where：意味着MySQL服务器将在存储引擎检索后再进行过滤；
3. Using temporary：意味着MySQL在对查询结果排序时会使用一个临时表；
4. Using filesort：意味着MySQL会对结果使用一个外部索引排序，而不是按索引次序从表里读取行；
5. Range checked for each record …：意味着没有好用的索引；

# 附录E 锁的调试

（略）

# 在MySQL上使用Sphinx

免费的开源的全文搜索引擎

# 参考

https://segmentfault.com/a/1190000011722724

https://blog.csdn.net/yxz8102/article/details/107391026

https://juejin.im/post/6844904061582442509

https://blog.csdn.net/xifeijian/category_1430276.html

