# mysql 锁介绍

最近遇见了几次死锁，所以准备写了一篇关于锁相关内容。



# base 最基础的概念

* shared lock  共享锁  

  许可：  事务 读取 a row（data item 数据项）。

* exclusive lock 徘他锁

  许可：  事务 读取、更新或者删除 a row <font color="red">官方文档上没有读取操作的关键词</font>

注意上面在教科书里面的提法是 shared-mode lock 和exclusive-mode lcok、共享型锁和排他型锁。

每个transcation 需要 根据自己对 data item 的操作类型发送申请（reqeust）获取适当的锁。请求发送给**并发控制管理器**（concurrency-control manager），只有被**授予**（grant）所需要的锁之后才能继续操作。



##  相容性表格 compatibility  table

|      | S    | X    |
| ---- | ---- | ---- |
| S    | true | F    |
| X    | F    | F    |

这个表展示在并发控制的时候加锁的行为，如果两个锁止键是不相容的，后来的申请获取锁进入等待。

## 死锁 deadlock 、活锁live lock 以及 饥饿 starved

### 死锁

```
　死锁的条件 仅供参考，说法应该是不严谨的
　1、禁止抢占：系统资源不能被强制从一个进程中退出。
　2、持有和等待：一个进程可以在等待时持有系统资源。
　3、互斥:资源只能同时分配给一个行程，无法多个行程共享。
　4、循环等待:一系列进程互相持有其他进程所需要的资源。
```

对于进程来说： 如果没有获取资源会陷入等待，获取的资源不会主动释放，多个进程之间拥有占有需要的资源。

对于资源来说： 只能分配个有限个进程，不能让所有的需要获取资源的进程都获取到。



![img](https://upload.wikimedia.org/wikipedia/commons/thumb/2/28/Process_deadlock.svg/220px-Process_deadlock.svg.png)

依赖图： p1拥有r2，请求p1资源。

### 活锁

与死锁相似，死锁是行程都在等待对方先释放资源。活锁则是行程彼此释放资源又同时占用对方释放的资源。当此情况持续发生时，尽管资源的状态不断改变，但每个行程都无法获取所需资源，使得事情没有任何进展。

### 饥饿

使用非公平锁导致的 transcation 永远等待的现象。

例如请求加锁的顺序如下：

 S S X S S 

但是我们可以看到 S 和S是相容的，可以变成

SSSS X  

这种情况下徘他锁可能永远都不能获取下去，称之为饥饿。

为了预防这种情况，对数据项申请M型锁时，并发控制管理器授权锁的条件为：

* 不存在在数据项Q上持有与M型锁冲突的锁的其他事务。

* <font color="red">不存在等待对数据项Q加锁且优先于当前事务加锁的事务。</font>



### 两阶段封锁协议 two - phase locking protocol

他要求事物分到两个阶段 提出加锁和解锁的申请：

1、  增长阶段 growing phase  这一阶段只能获取锁。

2、  缩减阶段 shrinking phase  这一阶段只能释放锁。

然后对这协议进行更多的约束得到两个更强的协议：

严格两阶段封锁协议 strict two phase locking protocol。所有的徘他锁只能在事务提交之后才可以释放。 可以用来保证 未提交的事物可不读。

强两阶段封锁协议rigorous two phase locking protocal。 所有的锁只能在事务提交之后才可以释放。这个是用来保证提交的顺序串行化.

#### 锁转换

在两段封锁协议下，可以实现锁转换（lock conversion）。在增长阶段，将S锁升级（upgrade）成X锁，在缩减阶段将X锁（downgrade）成S锁。



## 死锁的处理

### 死锁的预防

* 一次性提出所需要的所有锁，要么成功，要么失败。
* 对需要的锁加次序。
* 使用抢占preempt 和回滚rollback

​       wait-die 当前Ti申请的数据项被Tj占有，当且仅当 Ti的时间戳大于Tj。否则让Tj回滚。

​       wound-wait 当前Ti申请的数据项被Tj占有，当且仅当 Tj的时间戳大于Ti。否则让Tj回滚。

* 锁超时。 如果等待时间过短则没有死锁也会回滚。也会导致饿死

  

### 死锁的检测与恢复

基于最小代价rocover选择事务回滚。

* 该事务已经计算了多久，并且还需要计算多久。
* 使用了多少数据项。
* 还需要多少数据项。
* 回滚时牵扯多少事务。

回滚需要考虑事情：

   彻底回滚。

   部分回滚。

饿死 

​    如果某事物因为回滚导致总不能完成任务，就发生了饿死。

 <hr />

# 多粒度封锁协议 muyltiple-granularity locking protocol

## 意向锁 Intention Locks

意向锁是用来解决更高粒度上加锁。

在申请X或S之前，插入更大范围的意向锁。



innoDb支持多种粒度的锁。这些锁可以共存于行锁和表锁。

对于申请锁，可以在使用前插入意向锁，等真正需要的时候才进行加S和X锁。

附 主动加锁的语法

```sql
LOCK TABLES
    tbl_name [[AS] alias] lock_type
    [, tbl_name [[AS] alias] lock_type] ...

lock_type: {
    READ [LOCAL]
  | [LOW_PRIORITY] WRITE
}

UNLOCK TABLES
```

```
SELECT ... FOR SHARE
SELECT ... FOR UPDATE 
```

相容表

|      | X        | IX         | S          | IS         |
| :--- | :------- | :--------- | :--------- | ---------- |
| `X`  | Conflict | Conflict   | Conflict   | Conflict   |
| `IX` | Conflict | Compatible | Conflict   | Compatible |
| `S`  | Conflict | Conflict   | Compatible | Compatible |
| `IS` | Conflict | Compatible | Compatible | Compatible |



### 并发问题

| 脏写 dirty write               | 如果一个数据项被另一个尚未提交或终止的事务写入，则不允许对该数据项执行写操作 |
| ------------------------------ | ------------------------------------------------------------ |
| 脏读 dirty read                | 读取到了其他事务没有提交的数据。                             |
| 不可重复读 nonrepeatable reads | 当事务读取一条记录两次，但是两次获取到的是不同的数据。       |
| 幻读                           | 存在一个像幽灵一样的数据，在初始阶段匹配的时候没有看到，然后被看到。 |





# 在不同隔离级别下的 讨论

##  读未提交

### 记录锁 record locks

所有的隔离级别下面，都会使用记录锁。

该锁会针对对唯一性索引和主键的进行加锁。

```sql
select... from table 。。。 for  update。

select… from table …. lock share mode

update ..

insert ...

delete
```

给出一个会发生死锁的例子，保证了不会发生脏写。

```sql
t1:
begin;
select * from t where id =1 for update;
select * from t where id =2 for update;
commit;
t2:
begin;
select * from t where id =2 for update;
select * from t where id =1 for update;
commit;

```

接下来的sql展示了读未提交

| SET session TRANSACTION  ISOLATION LEVEL READ UNCOMMITTED; | SET session TRANSACTION  ISOLATION LEVEL READ UNCOMMITTED; |
| ---------------------------------------------------------- | ---------------------------------------------------------- |
| begin                                                      | begin                                                      |
| update t set c1 = 333 where id =3;                         |                                                            |
|                                                            | select * from t;                                           |
| rollback                                                   |                                                            |
|                                                            | select * from t;                                           |
|                                                            | commit                                                     |

### 乐观的并发控制机制

大部分事务都是只读的事物，发生冲突的频率很低。

有效性检查协议（validation protocol）要求每个事物Ti在生命周期按照两个或者三个阶段执行，这取决于事物是一个只读事务还是一个更新事务。这些阶段顺序如下：

- 读阶段（read phase） 这一阶段中，系统执行事务Ti。各数据项值被读入并保存在事务Ti的局部变量中。所有的write操作都是对局部临时变量进行中，并不对数据库进行真正的更新。
- 有效性检查阶段（validation phase）对事务Ti进行**有效性测试**。判断是否可以执行write操作而不违反串行性。如果事务有效性测试失败，则终止这个事务。
- 写阶段（write phase） 若事务Ti已通过有效性检查，则保存Ti任何写操作结果的临时局部变量值被复制到数据库中。只读事务忽略这个阶段。

定义：

- start（Ti） 事务Ti开始的执行时间。
- Validation（Ti） 事务完成读阶段并开始其有效性检查时间。
- Finish（Ti） 事务Ti完成写阶段的时间。

##### 进行有效性检查

满足TS(Tk)<TS(Ti)的事务Tk必须满足下面的两个条件之一：

- Finish（Tk) < Start(Ti)
- Tk所写入的数据项集于Ti所读数据项集不相交，并且Tk写阶段在Ti开始进行有效性有效性检查之前就完成（Start（Ti） < Finish(Tk)<Validation(Ti)) 。这个条件保证了Tk和Ti写不重叠。

乐观的并发控制当发生冲突的时候，会采用失败尝试的方式。可能导致长事务的饿死现象。

* 思考：然后我把条件2弱化。不要求 Tk所写入的数据项集于Ti所读数据项集不相交。是什么隔离级别？



## 读已提交

##### 多版本并发控制协议（ multiversion concurrency control） 

每一个write（Q）操作创建Q的一个新版本（version）。当事务发送Read（Q）操作的，并发控制管理器选择一个Q的版本就行读取。



对于每个数据项Q，有一个版本序列<Q1,Q2,…Qm>与之关联，每个版本Qk包含三个数据字段。

* content是Qk版本的值。
* w-timestamp（Q）是创建Qk版本的事物的时间戳。
* r-timestamp（Q） 是所有事成功读取Qk的事务的最大时间戳。

事务通过发送 write（Q）操作创建一个数据项Q的一个新版本Qk，版本的content字段保存事务Ti写入的值，系统将W-timestamp与R-timestamp初始化为 TS(Ti)。当事务Tj读取Qk的值切R-timestamp（Qk）<TS(Tj)时，R-timestamp的值就更新。

##### 多版本时间戳排序机制（multiversion timestamp-ordering scheme）保证串行化

假设事务Ti发出read（Q）或者write（Q）操作，另Qk表示Q满足如下条件的版本，其写时间戳是小于或等于TS（Ti）的最大写时间戳。

1、 如果事务Ti发出read（Q），则返回值就是Qk的内容。

2、如果事务Ti发出write（Q），且若TS（Ti）< R-timestamp(Qk),则系统回滚事务Ti，另一方面，若TS（Ti）= w-timestamp（Qk），则覆盖Qk的内容，否则 创建一个Q的新版本。

##### 多版本两阶段封锁协议

* 只读事务

  只读事务在开始执行之前，数据库系统读取ts-counter的当前值作为该事务的时间戳。只读事务在实行读操作的遵守多版本时间排序协议。因此，当只读事务Ti发送read（Q）时，返回的是小于当前TS（Ti）的最大时间戳的版本内容。

* 更新事务

  更新事务执行强两阶段封锁协议。当更新事务读取一个数据项时，他在获得该数据项上的共享锁后读取该数据项的最新版本的值。当更新事务想写一个数据项时，他首先要获取该数据项上的徘他锁，然后为此数据项创建一个新版本。写操作在新版本上时间戳设为无穷大。

  当更新事务Ti完成其任务的后，首先T将他创建的每一个版本的时间戳设制为ts-counter的值加1；然后Ti将ts-counter加1。同一时间内只能有一个更新事务进行提交。



### mysql innoDB 多版本

参考两个地方：需要参考两处 一致性读和多版本控制

 https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html

https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html

##### 一致性读

一致性读意味innoDB使用多版本来机制来在某个时间点给查询提供数据的快照。

他保证了在其他事务没有写操作没有提交之前，对当前事务的不可见。

在多版本控制下，事务的隔离级别

- 当事务隔离级别为REPEATABLE READ时，同一个事务中的一致性读都是读取的是该事务下第一次查询所建立的快照。
- 当事务隔离级别为READ COMMITTED时，同一事务下的一致性读都会建立和读取此查询自己的最新的快照

##### 多版本 

innodb 在每一行上插入三个隐藏列参考https://dev.mysql.com/doc/refman/8.0/en/innodb-row-format.html

| DB_ROW_ID   | 行ID，唯一标识一条记录，如果指定了primary key则不存在此字段 |
| ----------- | ----------------------------------------------------------- |
| DB_TRX_ID   | 事务ID                                                      |
| DB_ROLL_PTR | 回滚指针                                                    |



InnoDB 是一个数据多版本的存储引擎，它会保持它修改的数据的旧版本数据以此来支持事务特性，比如并发操作和事务的回滚。这些旧版本数据存储在一个叫做rollback segment的数据结构中（回滚段），当事务回滚的时候，Innodb会使用回滚段的数据来执行事务的撤销操作，也会使用这些老版本的数据来做旧版本的一致性读操作（可重复读的隔离级别下需要用到。 在读已提交隔离级别下，不需要老版本的数据来做一致性读。



回滚段中的undo log 分成insert 和update 两类，insert undo log 只在事务回滚中被需要，并且可以在事务提交时丢弃这类日志。update undo log被用于一致性读，只有当没有未提交的事务时才能被丢弃，Innodb为事务分配了一个快照在一致性读时，它需要update undo log的信息来构建数据的旧的版本数据。

Innodb多版本模式下，当你使用SQL语句删除行时它不会立刻从数据库物理删除数据，Innodb只会在丢弃相应的undo log时才会物理删除数据和索引数据，这个移除操作在Innodb中叫着purge，这个动作非常迅速，通常跟SQL的执行顺序一致。

如果插入和删除操作以差不多相同的速度进行，purge 线程会有延迟，表会变得越来越大因为表中的"dead" 行，这会导致性能被磁盘IO限制，在这种场景下，通过调整系统参数 [`innodb_max_purge_lag`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_max_purge_lag) 来分配跟多的资源给purge线程。

##### 多版本与二级索引

Innodb多版本并发控制对二级索引的处理不同于聚簇索引（primary key），聚簇索引中的记录会被就地更新，他们指向undo log的隐藏系统列可以构建旧版本数据，不同于聚簇索引，二级索引记录不包含隐藏系统列，不能被就地更新。

当二级索引被更新的时候，老的记录被标记删除，插入一条新记录，标记删除的数据最终被移除，当二级索引记录标记删除或者二级索引的页被一个新事务更新，Innodb在聚簇索引中找到这些记录，在聚簇索引中，检查这些行的 DB_TRX_ID，如果这些行在当前事务启动之后被更新了，那么就从undo log中获取老版本的数据。 

当二级索引记录被标记删除或者二级索引的页被一个新事务更新，索引覆盖技术不再适用，不同于直接从索引返回数据，Innodb会从聚簇索引中寻找数据。



## 可重复读 repeatable read

主要依靠的是多版本控制协议。上面已经进行了说明。mysql 在这个隔离级别下会使用间隙锁(gap lock)。间隙锁是主要是用来解决幻读问题的。

可以通过innodb_locks_unsafe_for_binlog = 1关闭间隙锁。

扩展 https://www.cnblogs.com/fanguangdexiaoyuer/p/11323248.html

#### 插入、删除与谓词读

##### 谓词读与幻读现象

```sql
select count(1) from t where c1 = 1;
会和下面两种sql起冲突
insert into t(c1) value(1);
以及 
update t set c1 = 1 where ...

```

原因都是使用的谓词读导致的问题。 可以通过下面的协议进行加锁解决。

##### 索引封锁协议（index-locking protocol）

* 每一个关系至少有一个索引。

* 只有首先在关系的一个或者多个索引上找到元祖后，事务Ti才能访问关系上的这些元祖。为了达到封锁协议的目的，全表扫描看作一个索引上所有叶结点的扫描
* 进行查找的事务Ti必须要在他要访问的所有索引叶结点上获取共享锁。
* 在没有更新关系r上的所有索引之前，事务Ti不能插入、删除或者更新r中的元祖ti。该事务必须获取插入、删除或更新锁影响的所有索引叶结点上的徘他锁。对于插入与删除，受影响的叶子结点是插入后包含或插入前包含元祖搜索码值的叶结点。对于更新，受影响的叶结点是修改前包含搜索码旧值的叶子结点以及修改后包含搜索码新值的叶子结点。
* 元祖总是获取锁。
* 必须遵守两阶段封锁协议。

英文原文：

* Every relation must have at least one index. 

*  A transaction *T**i* can access tuples of a relation only after fifirst fifinding them through one or more of the indices on the relation. For the purpose of the index-locking protocol, a relation scan is treated as a scan through all the leaves of one of the indices. 

*  A transaction *T**i* that performs a lookup (whether a range lookup or a point lookup) must acquire a shared lock on all the index leaf nodes that it accesses. 

* A transaction *T**i* may not insert, delete, or update a tuple *t**i* in a relation r without updating all indices on *r*. The transaction must obtain exclusive locks on all index leaf nodes that are affected by the insertion, deletion, or update. For insertion and deletion, the leaf nodes affected are those that contain (after insertion) or contained (before deletion) the search-key value of the tuple. For updates, the leaf nodes affected are those that (before the modifification) contained the old value of the search key, and nodes that (after the modifification) contain the new value of the search key. 

* Locks are obtained on tuples as usual. 

* The rules of the two-phase locking protocol must be observed.

##### 谓词锁

允许给谓词加共享锁。每次插入或更新都要检查是否满足谓词。这种协议称为 predicate locking。实践中不采用，因为它实现的代价很大。官方文档上写的gap锁更像是谓词锁，但是他并没有使用。实际上使用的是next key 锁。相比于谓词锁，他封锁的范围会稍大一些，因为他是基于索引的。



参考 数据库系统概念 15章

可以看出网上举出的例子很多都是 使用的普通查询 没有使用next key锁。进而引发幻读。



有关针对索引加锁的 可以参考下mysql innodb的存储结构

https://zhuanlan.zhihu.com/p/372830975