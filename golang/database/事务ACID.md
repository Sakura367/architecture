```
DDL(data defination language)数据库定义语言。常用的命令有：create，drop，alter，rename等等。
DML(data manipulation language)数据操作语言。常用的命令有：insert，update，delete等。
DCL(data control language)数据库控制语言。常用的命令有：grant，revoke  等。
DQL(data query language)数据查询语言。常用的命令有：select，order by，from， where等。
```

### 事务ACID
一致性：通过原子性、隔离性、持久性来保证一致性
原子性：undo log
持久性：redo log
隔离性：MVCC（多版本并发控制，实际上就是保存了数据在某个时间节点的快照）

- undo日志用于存放数据修改被修改前的值，如果修改出现异常，可以使用undo日志来实现回滚操作。
- redo日志用于顺序记录数据修改后的记录。当buffer pool中的dirty page 还没有刷新到磁盘的时候，发生crash，启动服务后，可通过redo日志找到需要重新刷新到磁盘文件的记录。
- binlog是记录所有数据库表结构变更（例如CREATE、ALTER TABLE…）以及表数据修改（INSERT、UPDATE、DELETE…）的二进制日志。
- redo日志由InnoDB引擎实现，是物理日志
- undo日志是逻辑日志，记录反向SQL操作
- binlog日志由MySQL的Server层实现，无关数据库引擎，是逻辑日志

### 事务执行过程
1. 先将原始数据从磁盘中读入内存中
2. 生成一条修改数据语句的反向操作日志到undo log
3. 修改数据的内存拷贝
4. 生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值
，并将该条 redo log 的状态标记为 prepare 状态
5. 接着存储引擎告诉执行器，可以提交事务了。执行器接到通知后，会写 binlog 日志，然后提交事务
6. 存储引擎接到提交事务的通知后，将 redo log 的日志状态标记为 commit 状态
7. 当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加写的方式
8. 根据需要删除undo log日志
9. 定期将内存中修改的数据刷新到磁盘中

**undo log的删除**
- 针对于insert undo log
insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。
- 针对于update undo log
undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

### MySQL宕机处理
#### redo log
MySQL 在更新数据时，为了减少磁盘的随机 IO，因此并不会直接更新磁盘上的数据，而是先更新 Buffer Pool 中缓存页的数据，等到合适的时间点，再将这个缓存页持久化到磁盘。而 Buffer Pool 中所有缓存页都是处于内存当中的，当 MySQL 宕机或者机器断电，内存中的数据就会丢失，因此 MySQL 为了防止缓存页中的数据在更新后出现数据丢失的现象，引入了 redo log 机制。

当进行增删改操作时，MySQL 会在更新 Buffer Pool 中的缓存页数据时，会记录一条对应操作的 redo log 日志，这样如果出现 MySQL 宕机或者断电时，如果有缓存页的数据还没来得及刷入磁盘，那么当 MySQL 重新启动时，可以根据 redo log 日志文件，进行数据重做，将数据恢复到宕机或者断电前的状态，保证了更新的数据不丢失，因此 redo log 又叫做重做日志。它的本质是保证事务提交后，更新的数据不丢失。

#### redo log buffer
当一条 SQL 更新完 Buffer Pool 中的缓存页后，就会记录一条 redo log 日志，前面提到了 redo log 日志是存储在磁盘上的，那么此时是不是立马就将 redo log 日志写入磁盘呢？显然不是的，而是先写入一个叫做 redo log buffer 的缓存中，redo log buffer 是一块不同于 buffer pool 的内存缓存区，在 MySQL 启动的时候，向内存中申请的一块内存区域，它是 redo log 日志缓冲区，默认大小是 16MB。

redo log buffer 内部又可以划分为许多 redo log block，每个 redo log block 大小为 512 字节。我们写入的 redo log 日志，最终实际上是先写入在 redo log buffer 的 redo log block 中，然后在某一个合适的时间点，将这条 redo log 所在的 redo log block 刷入到磁盘中。

这个合适的时间点究竟是什么时候呢？
1. MySQL 正常关闭的时候；
2. MySQL 的后台线程每隔一段时间定时的将 redo log buffer 刷入到磁盘，默认是每隔 1s 刷一次；
3. 当 redo log buffer 中的日志写入量超过 redo log buffer 内存的一半时，即超过 8MB 时，会触发 redo log buffer 的刷盘；
4. 当事务提交时，根据配置的参数 innodb_flush_log_at_trx_commit 来决定是否刷盘。如果 innodb_flush_log_at_trx_commit 参数配置为 0，表示事务提交时，不进行 redo log buffer 的刷盘操作；如果配置为 1，表示事务提交时，会将此时事务所对应的 redo log 所在的 redo log block 从内存写入到磁盘，同时调用 fysnc，确保数据落入到磁盘；如果配置为 2，表示只是将日志写入到操作系统的缓存，而不进行 fysnc 操作。（进程在向磁盘写入数据时，是先将数据写入到操作系统的缓存中：os cache，再调用 fsync 方法，才会将数据从 os cache 中刷新到磁盘上）

#### 保证数据不丢失
MySQL 究竟是如何来保证数据不丢失的。

MySQL Server 层的执行器调用 InnoDB 存储引擎的数据更新接口；
存储引擎更新 Buffer Pool 中的缓存页，
同时存储引擎记录一条 redo log 到 redo log buffer 中，并将该条 redo log 的状态标记为 prepare 状态；
接着存储引擎告诉执行器，可以提交事务了。执行器接到通知后，会写 binlog 日志，然后提交事务；
存储引擎接到提交事务的通知后，将 redo log 的日志状态标记为 commit 状态；
接着根据 innodb_flush_log_at_commit 参数的配置，决定是否将 redo log buffer 中的日志刷入到磁盘。
将 redo log 日志标记为 prepare 状态和 commit 状态，这种做法称之为两阶段事务提交，它能保证事务在提交后，数据不丢失。为什么呢？redo log 在进行数据重做时，只有读到了 commit 标识，才会认为这条 redo log 日志是完整的，才会进行数据重做，否则会认为这个 redo log 日志不完整，不会进行数据重做。

例如，如果在 redo log 处于 prepare 状态后，buffer pool 中的缓存页（脏页）也还没来得及刷入到磁盘，写完 biglog 后就出现了宕机或者断电，此时提交的事务是失败的，那么在 MySQL 重启后，进行数据重做时，在 redo log 日志中由于该事务的 redo log 日志没有 commit 标识，那么就不会进行数据重做，磁盘上数据还是原来的数据，也就是事务没有提交，这符合我们的逻辑。

实际上要严格保证数据不丢失，必须得保证 innodb_flush_log_at_trx_commit 配置为 1。

如果配置成 0，则 redo log 即使标记为 commit 状态了，由于此时 redo log 处于 redo log buffer 中，如果断电，redo log buffer 内存中的数据会丢失，此时如果恰好 buffer pool 中的脏页也还没有刷新到磁盘，而 redo log 也丢失了，所以在 MySQL 重启后，由于丢失了一条 redo log，因此就会丢失一条 redo log 对应的重做日志，这样断电前提交的那一次事务的数据也就丢失了。

如果配置成 2，则事务提交时，会将 redo log buffer（实际上是此次事务所对应的那条 redo log 所在的 redo log block ）写入磁盘，但是操作系统通常都会存在 os cache，所以这时候的写只是将数据写入到了 os cache，如果机器断电，数据依然会丢失。

而如果配置成 1，则表示事务提交时，就将对应的 redo log block 写入到磁盘，同时调用 fsync，fsync 会将数据强制从 os cache 中刷入到磁盘中，因此数据不会丢失。

从效率上来说，0 的效率最高，因为不涉及到磁盘 IO，但是会丢失数据；而 1 的效率最低，但是最安全，不会丢失数据。2 的效率居中，会丢失数据。在实际的生产环境中，通常要求是的是“双 1 配置”，即将 innodb_flush_log_at_trx_commit 设置为 1，另外一个 1 指的是写 binlog 时，将 sync_binlog 设置为 1，这样 binlog 的数据就不会丢失。

#### 宕机恢复
前面说到未提交的事务和回滚了的事务也会记录Redo Log，因此在进行恢复时,这些事务要进行特殊的的处理。有2种不同的恢复策略：

A. 进行恢复时，只重做已经提交了的事务。
B. 进行恢复时，重做所有事务包括未提交的事务和回滚了的事务。然后通过Undo Log回滚那些 未提交的事务。

MySQL数据库InnoDB存储引擎使用了B策略, InnoDB存储引擎中的恢复机制有几个特点：

A. 在重做Redo Log时，并不关心事务性。 恢复时，没有BEGIN，也没有COMMIT,ROLLBACK的行为。也不关心每个日志是哪个事务的。尽管事务ID等事务相关的内容会记入Redo Log，这些内容只是被当作要操作的数据的一部分。

B. 使用B策略就必须要将Undo Log持久化，而且必须要在写Redo Log之前将对应的Undo Log写入磁盘。Undo和Redo Log的这种关联，使得持久化变得复杂起来。为了降低复杂度，InnoDB将Undo Log看作数据，因此记录Undo Log的操作也会记录到redo log中。这样undo log就可以象数据一样缓存起来，而不用在redo log之前写入磁盘了。
包含Undo Log操作的Redo Log，看起来是这样的：
```
记录1: <trx1, Undo log insert <undo_insert …>>
记录2: <trx1, insert …>
记录3: <trx2, Undo log insert <undo_update …>>
记录4: <trx2, update …>
记录5: <trx3, Undo log insert <undo_delete …>>
记录6: <trx3, delete …>
```

C. 到这里，还有一个问题没有弄清楚。既然Redo没有事务性，那岂不是会重新执行被回滚了的事务？
确实是这样。同时Innodb也会将事务回滚时的操作也记录到redo log中。回滚操作本质上也是对数据进行修改，因此回滚时对数据的操作也会记录到Redo Log中。
一个回滚了的事务的Redo Log，看起来是这样的：
```
记录1: <trx1, Undo log insert <undo_insert …>>
记录2: <trx1, insert A…>
记录3: <trx1, Undo log insert <undo_update …>>
记录4: <trx1, update B…>
记录5: <trx1, Undo log insert <undo_delete …>>
记录6: <trx1, delete C…>
记录7: <trx1, insert C>
记录8: <trx1, update B to old value>
记录9: <trx1, delete A>
```

#### 必要性
首先引入 redo log 机制是十分必要的。因为写 redo log 时，我们将 redo log 日志追加到文件末尾，虽然也是一次磁盘 IO，但是这是顺序写操作（不需要移动磁头）；而对于直接将数据更新到磁盘，涉及到的操作是将 buffer pool 中缓存页写入到磁盘上的数据页上，由于涉及到寻找数据页在磁盘的哪个地方，这个操作发生的是随机写操作（需要移动磁头），相比于顺序写操作，磁盘的随机写操作性能消耗更大，花费的时间更长，因此 redo log 机制更优，能提升 MySQL 的性能。

从另一方面来讲，通常一次更新操作，我们往往只会涉及到修改几个字节的数据，而如果因为仅仅修改几个字节的数据，就将整个数据页写入到磁盘（无论是磁盘还是 buffer pool，他们管理数据的单位都是以页为单位），这个代价未免也太了（每个数据页默认是 16KB），而一条 redo log 日志的大小可能就只有几个字节，因此每次磁盘 IO 写入的数据量更小，那么耗时也会更短。

综合来看，redo log 机制的引入，在提高 MySQL 性能的同时，也保证了数据的可靠性。

#### binlog与redolog必要性
**有了redo log，为啥还需要binlog呢？**
1. redo log的大小是固定的，采用"循环滚动写"的方式记录log，日志上的记录修改落盘后，日志会被覆盖掉，无法用于全量数据的 "回滚/恢复"等操作。
2. redo log是innodb引擎层实现的，并不是所有引擎都有。

**有了binlog，为啥还需要redo log呢？**
1. redo log是从事务开始后逐步写入磁盘的，用于故障时，恢复已提交但未及时落盘写入数据盘的脏页数据（落盘的步骤是先从磁盘中把相关数据页读到内存中，然后更新内存中的数据页，更新后的内存数据页属于脏页【当前内存中的数据与磁盘中不一致】，之后脏页中的数据会被写入磁盘，称为刷脏页）；而 binlog是二进制日志，只有在事务即将提交时才写入（比如突然宕机了），因此没有保存脏页数据，并没有能力在数据库崩溃后恢复完整数据页（丢失部分未写入磁盘的数据）
2. 数据文件的检查点是由redolog的checkPoint记录的，大致可认为checkPoint之前的数据已经刷盘了，checkPoint之后的数据还没有刷盘，所以在数据库crash之后，redolog能够知道该从什么位置进行回放redolog日志进行恢复，而binlog却做不到

### MVCC
https://xie.infoq.cn/article/eff93ec47b54a5069e0bd1726

快照读：简单的select操作，属于快照读，不加锁。
select * from table where ?;

当前读：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。
select * from table where ? lock in share mode;
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;

**在快照读情况下，mysql通过mvcc来避免幻读。
在当前读情况下，mysql通过next-key来避免幻读。**

#### 隐式字段
InnoDB在每行数据都增加三个隐藏字段
- 唯一行号（DB_ROW_ID）
- 记录创建的版本号（DB_TRX_ID）
- 记录回滚的版本号（DB_ROLL_PTR）

#### undo log
undo log 细分为两种，insert 时产生的 undo log、update，delete 时产生的 undo log
- 在 Innodb 中 insert 产生的 undo log 在提交事务之后就会被删除，因为新插入的数据没有历史版本，所以无需维护 undo log。
- update 和 delete 操作产生的 undo log 都属于一种类型，在事务回滚时需要，而且在快照读时也需要，则需要维护多个版本信息。只有在快照读和事务回滚不涉及该日志时，对应的日志才会被purge线程统一删除。purge 线程会清理 undo log 的历史版本，同样也会清理 del  flag 标记的记录。

undo log 保存的是一个版本链，也就是使用 DB_ROLL_PTR 这个字段来连接的。

#### Read View
当执行 SQL 语句查询时会产生一致性视图，也就是 read-view，它是由查询的那一时间所有未提交事务 ID 组成的数组，和已经创建的最大事务 ID 组成的。

Read View 有四个重要的字段：
- m_ids ：指的是创建 Read View 时当前数据库中活跃且未提交的事务的事务 id 列表，注意是一个列表。
- min_trx_id ：指的是创建 Read View 时当前数据库中活跃且未提交的事务中最小事务的事务 id，也就是 m_ids 的最小值。
- max_trx_id ：这个并不是 m_ids 的最大值，而是创建 Read View 时当前数据库中应该给下一个事务的 id 值；
- creator_trx_id ：指的是创建该 Read View 的事务的事务 id。

#### 版本链对比规则
1. 数据行的 trx_id < min_trx_id，表示此版本是已经提交的事务生成的，由于事务已经提交所以数据是可见的
2. 数据行的 trx_id > max_trx_id，表示此版本是由将来启动的事务生成的，是肯定不可见的
3. 若在 min_trx_id <= trx_id <= max_trx_id 时
    1. 数据行的 的 trx_id 在 m_ids 数组中，表示此版本是由还没提交的事务生成的，不可见，但是当前自己的事务是可见的
    2. 数据行的 的 trx_id 不在 m_ids 数组中，表明是提交的事务生成了该版本，可见

还有一个特殊情况那就是对于已经删除的数据，在之前的 undo log 日志讲述时说了 update 和 delete 是同一种类型的 undo log，同样也可以认为 delete 就是 update 的特殊情况。

当删除一条数据时会将版本链上最新的数据复制一份，然后将 creator_trx_id 修改为删除时的 creator_trx_id delete flag 标记，将这个标记写上 true，用来表示当前记录已经删除。

在查询时按照版本链的规则查询到对应的记录，如果 delete flag 标记位为 true，意味着数据已经被删除，则不返回数据。

#### 读已提交与可重复读的Read View区别
- 读已提交隔离级别是在每个 select 都会生成一个新的 Read View，也意味着，事务期间的多次读取同一条数据，前后两次读的数据可能会出现不一致，因为可能这期间另外一个事务修改了该记录，并提交了事务。
- 可重复读隔离级别是启动事务时生成一个 Read View，然后整个事务期间都在用这个 Read View，这样就保证了在事务期间读到的数据都是事务启动前的记录。