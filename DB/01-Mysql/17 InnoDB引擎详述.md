**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员 MySQL数据库入门到精通，从mysql安装到mysql高级、mysql优化全囊括</font>**](https://www.bilibili.com/video/BV1Kr4y1i7ru/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

# 1 逻辑存储结构
InnoDB的逻辑存储结构如下图所示:  
![](images/219.png)

+ **表空间**：是InnoDB存储引擎逻辑结构的最高层， 如果用户启用了参数 innodb_file_per_table(在8.0版本中默认开启) ，**<font style="color:#DF2A3F;">则每张表都会有一个表空间（xxx.ibd），一个mysql实例可以对应多个表空间</font>**，用于存储记录、索引等数据。
+ **段**：分为<font style="color:#DF2A3F;">数据段</font>（Leaf node segment）、<font style="color:#DF2A3F;">索引段</font>（Non-leaf node segment）、<font style="color:#DF2A3F;">回滚段</font>（Rollback segment），InnoDB是索引组织表，<font style="color:#DF2A3F;">数据段就是B+树的叶子节点</font>， <font style="color:#DF2A3F;">索引段即为B+树的非叶子节点</font>。段用来管理多个Extent（区）。
+ **区**：表空间的单元结构，<font style="color:#DF2A3F;">每个区的大小为1M</font>。 默认情况下， <font style="color:#DF2A3F;">InnoDB存储引擎页大小为16K， 即一个区中一共有64个连续的页</font>。
+ **页**：是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16KB。为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4-5 个区。
+ **行**：InnoDB 存储引擎数据是按行进行存放的。在行中，默认有两个隐藏字段： 
    - Trx_id：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。
    - Roll_pointer：每次对某条引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

关于 ibd 文件

![](images/220.png)

# 2 架构
## 2.1 概述
MySQL 5.5 版本开始，默认使用InnoDB存储引擎，它擅长事务处理，具有崩溃恢复特性，在日常开发中使用非常广泛。下面是InnoDB架构图，左侧为内存结构，右侧为磁盘结构。

![](images/221.png)

## 2.2 内存结构
![](images/222.png)



在左侧的内存结构中，主要分为这么四大块儿：

+ Buffer Pool
+ Change Buffer
+ Adaptive Hash Index
+ Log Buffer



左边的一个个小方块就可以理解为一个个页

接下来介绍一下这四个部分。

**<font style="color:#E8323C;"></font>**

**<font style="color:#E8323C;">Buffer Pool</font>**

InnoDB存储引擎基于磁盘文件存储，访问物理硬盘和在内存中进行访问，速度相差很大，为了尽可能弥补这两者之间的I/O效率的差值，就需要把经常使用的数据加载到缓冲池中，避免每次访问都进行磁盘I/O。

在InnoDB的缓冲池中不仅缓存了索引页和数据页，还包含了undo页、插入缓存、自适应哈希索引以及InnoDB的锁信息等等。

缓冲池 Buffer Pool，是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，从而减少磁盘IO，加快处理速度。  
缓冲池以Page页为单位，底层采用链表数据结构管理Page。根据状态，将Page分为三种类型：

+ free page：空闲page，未被使用。
+ clean page：被使用page，数据没有被修改过。
+ dirty page：脏页，被使用page，数据被修改过，也中数据与磁盘的数据产生了不一致。

在专用服务器上，通常将多达80％的物理内存分配给缓冲池 。参数设置： `show variableslike 'innodb_buffer_pool_size';`



**<font style="color:#E8323C;">Change Buffer</font>**  
Change Buffer，更改缓冲区（针对于非唯一二级索引页），在执行DML（增删改）语句时，如果这些数据Page没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer中，在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再将合并后的数据刷新到磁盘中。  


Change Buffer的意义是什么呢?  
先来看一幅图，这个是二级索引的结构图：  
![](images/223.png)

与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引。同样，删除和更新可能会影响索引树中不相邻的二级索引页，如果每一次都操作磁盘，会造成大量的磁盘IO。有了ChangeBuffer之后，我们可以在缓冲池中进行合并处理，减少磁盘IO。



**<font style="color:#E8323C;">Adaptive Hash Index</font>**  
自适应hash索引，用于优化对Buffer Pool数据的查询。**MySQL的innoDB引擎中虽然没有直接支持hash索引**，但是给我们提供了一个功能就是这个自适应hash索引。因为前面我们讲到过，hash索引在进行等值匹配时，一般性能是要高于B+树的，因为hash索引一般只需要一次IO即可，而B+树，可能需要几次匹配，所以hash索引的效率要高，但是hash索引又不适合做范围查询、模糊匹配等。

InnoDB存储引擎会监控对表上各索引页的查询，如果观察到在特定的条件下hash索引可以提升速度，则建立hash索引，称之为自适应hash索引。

自适应哈希索引，无需人工干预，是系统根据情况自动完成。

参数： adaptive_hash_index

![](images/224.png)

ON代表的是开启。



**<font style="color:#E8323C;">Log Buffer</font>**  
Log Buffer：日志缓冲区，用来保存要写入到磁盘中的log日志数据（redo log 、undo log），默认大小为 16MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或删除许多行的事务，增加日志缓冲区的大小可以节省磁盘 I/O。

参数:

+ innodb_log_buffer_size：缓冲区大小

![](images/225.png)

+ innodb_flush_log_at_trx_commit：日志刷新到磁盘时机，取值主要包含以下三个：

![](images/226.png)

    - 1：日志在每次事务提交时写入并刷新到磁盘，默认值。
    - 0：每秒将日志写入并刷新到磁盘一次。
    - 2：日志在每次事务提交后写入，并每秒刷新到磁盘一次。  


## 2.3 磁盘结构
接下来，再来看看InnoDB体系结构的右边部分，也就是磁盘结构：  
![](images/227.png)  
**<font style="color:#E8323C;">System Tablespace</font>**  
系统表空间是更改缓冲区的存储区域。如果表是在系统表空间而不是每个表文件或通用表空间中创建的，它也可能包含表和索引数据。(在MySQL5.x版本中还包含InnoDB数据字典、undolog等)

参数：innodb_data_file_path  
![](images/228.png)  
系统表空间，默认的文件名叫 ibdata1。

![](images/229.png)

由于我们的mysql是docker创建的

```bash
docker run -d -p 3306:3306 -v /Users/admin/Desktop/docker-container/mysql/conf:/etc/mysql/conf.d -v /Users/admin/Desktop/docker-container/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 --name mysql mysql:latest
```

此文件原在 /var/lib/mysql 目录下，被我们挂载映射到 /Users/admin/Desktop/docker-container/mysql/data下，我们来看看是不是这个目录。

![](images/230.png)

  
**<font style="color:#E8323C;">File-Per-Table Tablespaces</font>**  
如果开启了innodb_file_per_table开关 ，则**每个表的文件表空间包含单个InnoDB表的数据和索引 **，并存储在文件系统上的单个数据文件中。  
开关参数：innodb_file_per_table ，该参数默认开启。  
![](images/231.png)  
那也就是说，我们每创建一个表，都会产生一个表空间文件，如图：  
![](images/232.png)

  
**<font style="color:#E8323C;">General Tablespaces</font>**  
通用表空间，需要通过 CREATE TABLESPACE 语法创建通用表空间，在创建表时，可以指定该表空间。

1. 创建表空间

```plsql
CREATE TABLESPACE ts_name ADD DATAFILE 'file_name' ENGINE = engine_name;
```

![](images/233.png)

看看我们创立的表空间

![](images/234.png)

2. 创建表时指定表空间

```plsql
CREATE TABLE xxx ... TABLESPACE ts_name;
```

![](images/235.png)

**<font style="color:#E8323C;"></font>**

**<font style="color:#E8323C;">Undo Tablespaces</font>**

撤销表空间：MySQL实例在初始化时会自动创建两个默认的undo表空间（初始大小16M），用于存储undo log日志。

![](images/236.png)



**<font style="color:#E8323C;">Temporary Tablespaces：临时表空间</font>**  
InnoDB 使用会话临时表空间和全局临时表空间。存储用户创建的临时表等数据。



**<font style="color:#E8323C;">Doublewrite Buffer Files</font>**

双写缓冲区：innoDB引擎将数据页从Buffer Pool刷新到磁盘前，先将数据页写入双写缓冲区文件中，便于系统异常时恢复数据。  
![](images/237.png)

**<font style="color:#E8323C;"></font>**

**<font style="color:#E8323C;">Redo Log</font>**

重做日志：是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log），前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都会存到该日志中，用于在刷新脏页到磁盘时,发生错误时, 进行数据恢复使用。

以循环方式写入重做日志文件，涉及两个文件：  
![](images/238.png)  
前面我们介绍了InnoDB的内存结构，以及磁盘结构，那么内存中我们所更新的数据，又是如何到磁盘中的呢？ 此时，就涉及到一组后台线程，接下来，就来介绍一些InnoDB中涉及到的后台线程。  
![](images/239.png)

##  2.4 后台线程
![](images/240.png)

在InnoDB的后台线程中，分为4类，分别是：

+ Master Thread
+ IO Thread
+ Purge Thread
+ Page Cleaner Thread

**<font style="color:#E8323C;"></font>**

**<font style="color:#E8323C;">Master Thread</font>**  
核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中, 保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo页的回收 。



**<font style="color:#E8323C;">IO Thread</font>**  
在InnoDB存储引擎中大量使用了AIO来处理IO请求, 这样可以极大地提高数据库的性能，而IOThread主要负责这些IO请求的回调。

| 线程类型 | 默认个数 | 职责 |
| --- | --- | --- |
| Read thread | 4 | 负责读操作 |
| Write thread | 4 | 负责写操作 |
| Log thread | 1 | 负责将日志缓冲区刷新到磁盘 |
| Insert buffer thread | 1 | 负责将写缓冲区内容刷新到磁盘 |


我们可以通过以下的这条指令，查看到InnoDB的状态信息，其中就包含IO Thread信息。

```sql
show engine innodb status \G;
```

![](images/241.png)

可以看到全部采用的是aio，是异步io。

**<font style="color:#E8323C;">Purge Thread</font>**  
主要用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收。

  
**<font style="color:#E8323C;">Page Cleaner Thread</font>**  
协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞。



# 3 事务原理
## 3.1 事务基础
事务：事务是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，即**这些操作要么同时成功，要么同时失败**。

  
事务特性

+ 原子性（Atomicity）：事务是不可分割的最小操作单元，要么全部成功，要么全部失败。
+ 一致性（Consistency）：事务完成时，必须使所有的数据都保持一致状态。
+ 隔离性（Isolation）：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行。
+ 持久性（Durability）：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。



那实际上，我们研究事务的原理，就是研究MySQL的InnoDB引擎是如何保证事务的这四大特性的。  
![](images/242.png)



而对于这四大特性，实际上分为两个部分。 其中的原子性、一致性、持久化，实际上是由InnoDB中的两份日志来保证的，一份是redo log日志，一份是undo log日志。 而持久性是通过数据库的锁，加上MVCC来保证的。  
![](images/243.png)  
我们在讲解事务原理的时候，主要就是来研究一下redolog，undolog以及MVCC。

## 3.2 redo log
重做日志，记录的是事务提交时数据页的物理修改，**是用来实现事务的****<font style="color:#DF2A3F;">持久性</font>**。

该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log file），前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都存到该日志文件中, 用于在刷新脏页到磁盘,发生错误时, 进行数据恢复使用。  


如果没有redolog，可能会存在什么问题的？ 我们一起来分析一下。

我们知道，在InnoDB引擎中的内存结构中，主要的内存区域就是缓冲池，在缓冲池中缓存了很多的数据页。 当我们在一个事务中，执行多个增删改的操作时，InnoDB引擎会先操作缓冲池中的数据，如果缓冲区没有对应的数据，会通过后台线程将磁盘中的数据加载出来，存放在缓冲区中，然后将缓冲池中的数据修改，修改后的数据页我们称为脏页。 而脏页则会在一定的时机，通过后台线程刷新到磁盘中，从而保证缓冲区与磁盘的数据一致。 而缓冲区的脏页数据并不是实时刷新的，而是一段时间之后将缓冲区的数据刷新到磁盘中，假如刷新到磁盘的过程出错了，而提示给用户事务提交成功，而数据却没有持久化下来，这就出现问题了，没有保证事务的持久性。  
![](images/244.png)

那么，如何解决上述的问题呢？ 在InnoDB中提供了一份日志 redo log，接下来我们再来分析一下，通过redolog如何解决这个问题。  
![](images/245.png)  
有了 redolog 之后，当对缓冲区的数据进行增删改之后，会首先将操作的数据页的变化，记录在redo log buffer中。在事务提交时，会将redo log buffer中的数据刷新到redo log磁盘文件中。过一段时间之后，如果刷新缓冲区的脏页到磁盘时，发生错误，此时就可以借助于redo log进行数据恢复，这样就保证了事务的持久性。 而如果脏页成功刷新到磁盘或或者涉及到的数据已经落盘，此时redolog就没有作用了，就可以删除了，所以存在的两个redolog文件是循环写的。

那为什么每一次提交事务，要刷新redo log 到磁盘中呢，而不是直接将buffer pool中的脏页刷新到磁盘呢 ?  
因为在业务操作中，我们操作数据一般都是随机读写磁盘的，而不是顺序读写磁盘。 而redo log在往磁盘文件中写入数据，由于是日志文件，所以都是顺序写的。顺序写的效率，要远大于随机写。 这种先写日志的方式，称之为 WAL（Write-Ahead Logging）。

## 3.3 undo log
**回滚日志**：用于记录数据被修改前的信息 , 作用包含两个 : 提供回滚(保证事务的原子性) 和MVCC(多版本并发控制) 。用于解决原子性问题。

undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。

**Undo log销毁**：undo log在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些日志可能还用于MVCC。

**Undo log存储**：undo log采用段的方式进行管理和记录，存放在前面介绍的 rollback segment回滚段中，内部包含1024个undo log segment。

# 4 MVCC
## 4.1 基本概念
**当前读**：指的是读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。对于我们日常的操作，如：select ... lock in share mode(共享锁)，select ...for update、update、insert、delete(排他锁)都是一种当前读。

测试：

```sql
create table stu(
    id int primary key auto_increment,
    name varchar(32),
    age int
);
insert into stu(name,age) value("java",21);
insert into stu(name,age) value("h5",28);
```

![](images/246.png)

1. 开启事务A
2. 开启事务B
3. 在事务A中查询stu表的所有数据
4. 在事务B中修改stu表中id为1的数据
5. 在事务A中查询stu表的所有数据，发现id为1的数据没有变化
6. 在事务B中提交本次事务
7. 在事务A中查询stu表的所有数据，发现id为1的数据没有变化
8. 在事务A中查询stu表的所有数据，加上lock in share mode(共享锁)，发现id为1的数据是最新数据

在测试中我们可以看到，即使是在默认的RR隔离级别下，事务A中依然可以读取到事务B最新提交的内容，因为在查询语句后面加上了 lock in share mode 共享锁，此时是当前读操作。当然，当我们加排他锁的时候，也是当前读操作。

![](images/247.png)



**快照读**：简单的select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据，不加锁，是非阻塞读。

![](images/248.png)

+ Read Committed：每次select，都生成一个快照读。
+ Repeatable Read：开启事务后第一个select语句才是快照读的地方。

![](images/249.png)

+ Serializable：快照读会退化为当前读。

测试  
![](images/250.png)  
在测试中,我们看到即使事务B提交了数据,事务A中也查询不到。 原因就是因为普通的select是快照读，而在当前默认的RR隔离级别下，开启事务后第一个select语句才是快照读的地方，后面执行相同的select语句都是从快照中获取数据，可能不是当前的最新数据，这样也就保证了可重复读。



MVCC：全称 Multi-Version Concurrency Control，多版本并发控制。指维护一个数据的多个版本，使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC的具体实现，还需要依赖于数据库记录中的三个隐式字段、undo log日志、readView。

接下来，我们再来介绍一下InnoDB引擎的表中涉及到的隐藏字段 、undolog 以及 readview，从而来介绍一下MVCC的原理。

## 4.2 隐藏字段
### 4.2.1 介绍
![](images/251.png)

当我们创建了上面的这张表，我们在查看表结构的时候，就可以显式的看到这三个字段。 实际上除了这三个字段以外，InnoDB还会自动的给我们添加三个隐藏字段及其含义分别是：

| 隐藏字段 | 含义 |
| --- | --- |
| DB_TRX_ID | 最近修改事务ID，记录插入这条记录或最后一次修改该记录的事务ID。 |
| DB_ROLL_PTR | 回滚指针，指向这条记录的上一个版本，用于配合undo log，指向上一个版本。 |
| DB_ROW_ID | 隐藏主键，如果表结构没有指定主键，将会生成该隐藏字段。 |


而上述的前两个字段是肯定会添加的， 是否添加最后一个字段DB_ROW_ID，得看当前表有没有主键，如果有主键，则不会添加该隐藏字段。

### 4.2.2 测试
**查看有主键的表 stu**  
进入服务器中的 /Users/admin/Desktop/docker-container/mysql/data/db2019 , 查看stu的表结构信息, 通过如下指令:

```graphql
ibd2sdi stu.ibd
```

查看到的表结构信息中，有一栏 columns，在其中我们会看到处理我们建表时指定的字段以外，还有额外的两个字段 分别是：DB_TRX_ID 、 DB_ROLL_PTR ，因为该表有主键，所以没有DB_ROW_ID隐藏字段。

```json
admin@cg db2019 % ibd2sdi stu.ibd
["ibd2sdi"
,
{
	"type": 1,
	"id": 398,
	"object":
		{
    "mysqld_version_id": 80032,
    "dd_version": 80023,
    "sdi_version": 80019,
    "dd_object_type": "Table",
    "dd_object": {
        "name": "stu",
        "mysql_version_id": 80032,
        "created": 20230510025716,
        "last_altered": 20230510025716,
        "hidden": 1,
        "options": "avg_row_length=0;encrypt_type=N;key_block_size=0;keys_disabled=0;pack_record=1;stats_auto_recalc=0;stats_sample_pages=0;",
        "columns": [
            {
                "name": "id",
                "type": 4,
                "is_nullable": false,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": true,
                "is_virtual": false,
                "hidden": 1,
                "ordinal_position": 1,
                "char_length": 11,
                "numeric_precision": 10,
                "numeric_scale": 0,
                "numeric_scale_null": false,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": false,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "AAAAAA==",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "interval_count=0;",
                "se_private_data": "table_id=1091;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 2,
                "column_type_utf8": "int",
                "elements": [],
                "collation_id": 255,
                "is_explicit_collation": false
            },
            {
                "name": "name",
                "type": 16,
                "is_nullable": true,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 1,
                "ordinal_position": 2,
                "char_length": 128,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "interval_count=0;",
                "se_private_data": "table_id=1091;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "varchar(32)",
                "elements": [],
                "collation_id": 255,
                "is_explicit_collation": false
            },
            {
                "name": "age",
                "type": 4,
                "is_nullable": true,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 1,
                "ordinal_position": 3,
                "char_length": 11,
                "numeric_precision": 10,
                "numeric_scale": 0,
                "numeric_scale_null": false,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "interval_count=0;",
                "se_private_data": "table_id=1091;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "int",
                "elements": [],
                "collation_id": 255,
                "is_explicit_collation": false
            },
            {
                "name": "DB_TRX_ID",
                "type": 10,
                "is_nullable": false,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 2,
                "ordinal_position": 4,
                "char_length": 6,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "",
                "se_private_data": "table_id=1091;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "",
                "elements": [],
                "collation_id": 63,
                "is_explicit_collation": false
            },
            {
                "name": "DB_ROLL_PTR",
                "type": 9,
                "is_nullable": false,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 2,
                "ordinal_position": 5,
                "char_length": 7,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "",
                "se_private_data": "table_id=1091;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "",
                "elements": [],
                "collation_id": 63,
                "is_explicit_collation": false
            }
        ],
        "schema_ref": "db2019",
        "se_private_id": 1091,
        "engine": "InnoDB",
        "last_checked_for_upgrade_version_id": 0,
        "comment": "",
        "se_private_data": "autoinc=0;version=0;",
        "engine_attribute": "",
        "secondary_engine_attribute": "",
        "row_format": 2,
        "partition_type": 0,
        "partition_expression": "",
        "partition_expression_utf8": "",
        "default_partitioning": 0,
        "subpartition_type": 0,
        "subpartition_expression": "",
        "subpartition_expression_utf8": "",
        "default_subpartitioning": 0,
        "indexes": [
            {
                "name": "PRIMARY",
                "hidden": false,
                "is_generated": false,
                "ordinal_position": 1,
                "comment": "",
                "options": "flags=0;",
                "se_private_data": "id=202;root=4;space_id=25;table_id=1091;trx_id=6940;",
                "type": 1,
                "algorithm": 2,
                "is_algorithm_explicit": false,
                "is_visible": true,
                "engine": "InnoDB",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "elements": [
                    {
                        "ordinal_position": 1,
                        "length": 4,
                        "order": 2,
                        "hidden": false,
                        "column_opx": 0
                    },
                    {
                        "ordinal_position": 2,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 3
                    },
                    {
                        "ordinal_position": 3,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 4
                    },
                    {
                        "ordinal_position": 4,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 1
                    },
                    {
                        "ordinal_position": 5,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 2
                    }
                ],
                "tablespace_ref": "db2019/stu"
            }
        ],
        "foreign_keys": [],
        "check_constraints": [],
        "partitions": [],
        "collation_id": 255
    }
}
}
,
{
	"type": 2,
	"id": 30,
	"object":
		{
    "mysqld_version_id": 80032,
    "dd_version": 80023,
    "sdi_version": 80019,
    "dd_object_type": "Tablespace",
    "dd_object": {
        "name": "db2019/stu",
        "comment": "",
        "options": "autoextend_size=0;encryption=N;",
        "se_private_data": "flags=16417;id=25;server_version=80032;space_version=1;state=normal;",
        "engine": "InnoDB",
        "engine_attribute": "",
        "files": [
            {
                "ordinal_position": 1,
                "filename": "./db2019/stu.ibd",
                "se_private_data": "id=25;"
            }
        ]
    }
}
}
]

```

**查看没有主键的表 employee**  
建表语句：

```plsql
create table employee (id int , name varchar(10));
```

此时，我们再通过以下指令来查看表结构及其其中的字段信息：

```plsql
ibd2sdi employee.ibd
```

查看到的表结构信息中，有一栏 columns，在其中我们会看到处理我们建表时指定的字段以外，还有额外的三个字段 分别是：DB_TRX_ID 、 DB_ROLL_PTR 、DB_ROW_ID，因为employee表是没有指定主键的。

```json
admin@cg db2019 % ibd2sdi employee.ibd
["ibd2sdi"
,
{
	"type": 1,
	"id": 400,
	"object":
		{
    "mysqld_version_id": 80032,
    "dd_version": 80023,
    "sdi_version": 80019,
    "dd_object_type": "Table",
    "dd_object": {
        "name": "employee",
        "mysql_version_id": 80032,
        "created": 20230511054438,
        "last_altered": 20230511054438,
        "hidden": 1,
        "options": "avg_row_length=0;encrypt_type=N;key_block_size=0;keys_disabled=0;pack_record=1;stats_auto_recalc=0;stats_sample_pages=0;",
        "columns": [
            {
                "name": "id",
                "type": 4,
                "is_nullable": true,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 1,
                "ordinal_position": 1,
                "char_length": 11,
                "numeric_precision": 10,
                "numeric_scale": 0,
                "numeric_scale_null": false,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "interval_count=0;",
                "se_private_data": "table_id=1092;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "int",
                "elements": [],
                "collation_id": 255,
                "is_explicit_collation": false
            },
            {
                "name": "name",
                "type": 16,
                "is_nullable": true,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 1,
                "ordinal_position": 2,
                "char_length": 40,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "interval_count=0;",
                "se_private_data": "table_id=1092;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "varchar(10)",
                "elements": [],
                "collation_id": 255,
                "is_explicit_collation": false
            },
            {
                "name": "DB_ROW_ID",
                "type": 10,
                "is_nullable": false,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 2,
                "ordinal_position": 3,
                "char_length": 6,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "",
                "se_private_data": "table_id=1092;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "",
                "elements": [],
                "collation_id": 63,
                "is_explicit_collation": false
            },
            {
                "name": "DB_TRX_ID",
                "type": 10,
                "is_nullable": false,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 2,
                "ordinal_position": 4,
                "char_length": 6,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "",
                "se_private_data": "table_id=1092;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "",
                "elements": [],
                "collation_id": 63,
                "is_explicit_collation": false
            },
            {
                "name": "DB_ROLL_PTR",
                "type": 9,
                "is_nullable": false,
                "is_zerofill": false,
                "is_unsigned": false,
                "is_auto_increment": false,
                "is_virtual": false,
                "hidden": 2,
                "ordinal_position": 5,
                "char_length": 7,
                "numeric_precision": 0,
                "numeric_scale": 0,
                "numeric_scale_null": true,
                "datetime_precision": 0,
                "datetime_precision_null": 1,
                "has_no_default": false,
                "default_value_null": true,
                "srs_id_null": true,
                "srs_id": 0,
                "default_value": "",
                "default_value_utf8_null": true,
                "default_value_utf8": "",
                "default_option": "",
                "update_option": "",
                "comment": "",
                "generation_expression": "",
                "generation_expression_utf8": "",
                "options": "",
                "se_private_data": "table_id=1092;",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "column_key": 1,
                "column_type_utf8": "",
                "elements": [],
                "collation_id": 63,
                "is_explicit_collation": false
            }
        ],
        "schema_ref": "db2019",
        "se_private_id": 1092,
        "engine": "InnoDB",
        "last_checked_for_upgrade_version_id": 0,
        "comment": "",
        "se_private_data": "",
        "engine_attribute": "",
        "secondary_engine_attribute": "",
        "row_format": 2,
        "partition_type": 0,
        "partition_expression": "",
        "partition_expression_utf8": "",
        "default_partitioning": 0,
        "subpartition_type": 0,
        "subpartition_expression": "",
        "subpartition_expression_utf8": "",
        "default_subpartitioning": 0,
        "indexes": [
            {
                "name": "PRIMARY",
                "hidden": true,
                "is_generated": false,
                "ordinal_position": 1,
                "comment": "",
                "options": "",
                "se_private_data": "id=203;root=4;space_id=26;table_id=1092;trx_id=7444;",
                "type": 2,
                "algorithm": 2,
                "is_algorithm_explicit": false,
                "is_visible": true,
                "engine": "InnoDB",
                "engine_attribute": "",
                "secondary_engine_attribute": "",
                "elements": [
                    {
                        "ordinal_position": 1,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 2
                    },
                    {
                        "ordinal_position": 2,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 3
                    },
                    {
                        "ordinal_position": 3,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 4
                    },
                    {
                        "ordinal_position": 4,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 0
                    },
                    {
                        "ordinal_position": 5,
                        "length": 4294967295,
                        "order": 2,
                        "hidden": true,
                        "column_opx": 1
                    }
                ],
                "tablespace_ref": "db2019/employee"
            }
        ],
        "foreign_keys": [],
        "check_constraints": [],
        "partitions": [],
        "collation_id": 255
    }
}
}
,
{
	"type": 2,
	"id": 31,
	"object":
		{
    "mysqld_version_id": 80032,
    "dd_version": 80023,
    "sdi_version": 80019,
    "dd_object_type": "Tablespace",
    "dd_object": {
        "name": "db2019/employee",
        "comment": "",
        "options": "autoextend_size=0;encryption=N;",
        "se_private_data": "flags=16417;id=26;server_version=80032;space_version=1;state=normal;",
        "engine": "InnoDB",
        "engine_attribute": "",
        "files": [
            {
                "ordinal_position": 1,
                "filename": "./db2019/employee.ibd",
                "se_private_data": "id=26;"
            }
        ]
    }
}
}
]
```

## 4.3 undolog
### 4.3.1 介绍
回滚日志：在insert、update、delete的时候产生的便于数据回滚的日志。

当insert的时候，产生的undo log日志只在回滚时需要，在事务提交后，可被立即删除。

而update、delete的时候，产生的undo log日志不仅在回滚时需要，在快照读时也需要，不会立即被删除。

### 4.3.2 undolog 版本链
有一张表原始数据为：  
![](images/252.png)

DB_TRX_ID : 代表最近修改事务ID，记录插入这条记录或最后一次修改该记录的事务ID，是自增的。  
DB_ROLL_PTR ： 由于这条数据是才插入的，没有被更新过，所以该字段值为null。



然后，有四个并发事务同时在访问这张表。

1. 第一步  
![](images/253.png)  
当事务2执行第一条修改语句时，会记录undo log日志，记录数据变更之前的样子; 然后更新记录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。  
![](images/254.png)
2. 第二步  
![](images/255.png)  
当事务3执行第一条修改语句时，也会记录undo log日志，记录数据变更之前的样子; 然后更新记录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。  
![](images/256.png)
3. 第三步  
![](images/257.png)  
当事务4执行第一条修改语句时，也会记录undo log日志，记录数据变更之前的样子; 然后更新记录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。  
![](images/258.png)

最终我们发现，不同事务或相同事务对同一条记录进行修改，会导致该记录的undolog生成一条记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录。

## 4.4 readview
ReadView（读视图）是 快照读 SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务（未提交的）id。  
ReadView中包含了四个核心字段：

| 字段 | 含义 |
| --- | --- |
| m_ids | 当前活跃的事务ID集合 |
| min_trx_id | 最小活跃事务ID |
| max_trx_id | 预分配事务ID，当前最大事务ID+1（因为事务ID是自增的） |
| creator_trx_id | ReadView创建者的事务ID |




而在readview中就规定了版本链数据的访问规则：

| 条件 | 是否可以访问 | 说明 |
| --- | --- | --- |
| trx_id == creator_trx_id | 可以访问该版本 | 成立，说明数据是当前这个事务更改的。 |
| trx_id < min_trx_id | 可以访问该版本 | 成立，说明数据已经提交了。 |
| trx_id > max_trx_id | 不可以访问该版本 | 成立，说明该事务是在ReadView生成后才开启。 |
| min_trx_id <= trx_id <= max_trx_id | 如果trx_id不在m_ids中，是可以访问该版本的 | 成立，说明数据已经提交。 |


不同的隔离级别，生成ReadView的时机不同：

+ READ COMMITTED ：在事务中每一次执行快照读时生成ReadView。
+ REPEATABLE READ：仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。

## 4.5 原理分析
### 4.5.1 RC隔离级别
RC隔离级别下，在事务中每一次执行快照读时生成ReadView。

我们就来分析事务5中，两次快照读读取数据，是如何获取数据的?

在事务5中，查询了两次id为30的记录，由于隔离级别为Read Committed，所以每一次进行快照读都会生成一个ReadView，那么两次生成的ReadView如下。  
![](images/259.png)  
那么这两次快照读在获取数据时，就需要根据所生成的ReadView以及ReadView的版本链访问规则，到undolog版本链中匹配数据，最终决定此次快照读返回的数据。

1. 先来看第一次快照读具体的读取过程：  
![](images/260.png)  
在进行匹配时，会从undo log的版本链，从上到下进行挨个匹配：

先匹配  
![](images/261.png)  
这条记录，这条记录对应的trx_id为4，也就是将4带入右侧的匹配规则中。 ①不满足 ②不满足 ③不满足 ④也不满足 ，都不满足，则继续匹配undo log版本链的下一条。  
再匹配第二条  
![](images/262.png)  
这条记录对应的trx_id为3，也就是将3带入右侧的匹配规则中。①不满足 ②不满足 ③不满足 ④也不满足 ，都不满足，则继续匹配undo log版本链的下一条  
再匹配第三条  
![](images/263.png)  
这条记录对应的trx_id为2，也就是将2带入右侧的匹配规则中。①不满足 ②满足 终止匹配，此次快照读，返回的数据就是版本链中记录的这条数据。



2. 再来看第二次快照读具体的读取过程:  
![](images/264.png)  
在进行匹配时，会从undo log的版本链，从上到下进行挨个匹配：  
先匹配  
![](images/265.png)  
这条记录，这条记录对应的trx_id为4，也就是将4带入右侧的匹配规则中。 ①不满足 ②不满足 ③不满足 ④也不满足 ，都不满足，则继续匹配undo log版本链的下一条。  
再匹配第二条  
![](images/266.png)  
这条记录对应的trx_id为3，也就是将3带入右侧的匹配规则中。①不满足 ②满足 。终止匹配，此次快照读，返回的数据就是版本链中记录的这条数据。

### 4.5.2 RR隔离级别
RR隔离级别下，仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。 而RR 是可重复读，在一个事务中，执行两次相同的select语句，查询到的结果是一样的。  
那MySQL是如何做到可重复读的呢? 我们简单分析一下就知道了  
![](images/267.png)  
我们看到，在RR隔离级别下，只是在事务中第一次快照读时生成ReadView，后续都是复用该ReadView，那么既然ReadView都一样， ReadView的版本链匹配规则也一样， 那么最终快照读返回的结果也是一样的。

所以呢，MVCC的实现原理就是通过 InnoDB表的隐藏字段、UndoLog 版本链、ReadView来实现的。而MVCC + 锁，则实现了事务的隔离性。 而一致性则是由redolog 与 undolog保证。



![](images/268.png)

