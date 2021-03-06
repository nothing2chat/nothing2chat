


# 拾人牙慧这个好像说的比较浅显

[文章链接参考](https://www.jb51.net/article/161042.htm)





####  事务的四个特性(acid): 
    原子性（Atomicity）、
    一致性（Consistency）、
    隔离性（Isolation）、
    持久性（Durability）

窃以为如果把mysql如何在实现这四个特性，应该就可以把mysql事务的具体实现说清楚了哈


# 问题一：Mysql怎么保证一致性的？
OK，这个问题分为两个层面来说。
从数据库层面，数据库通过原子性、隔离性、持久性来保证一致性。也就是说ACID四大特性之中，C(一致性)是目的，A(原子性)、I(隔离性)、D(持久性)是手段，是为了保证一致性，数据库提供的手段。数据库必须要实现AID三大特性，才有可能实现一致性。例如，原子性无法保证，显然一致性也无法保证。

但是，如果你在事务里故意写出违反约束的代码，一致性还是无法保证的。例如，你在转账的例子中，你的代码里故意不给B账户加钱，那一致性还是无法保证。因此，还必须从应用层角度考虑。

从应用层面，通过代码判断数据库数据是否有效，然后决定回滚还是提交数据！

# 问题二: Mysql怎么保证原子性的？
OK，是利用Innodb的undo log。
undo log名为回滚日志，是实现原子性的关键，当事务回滚时能够撤销所有已经成功执行的sql语句，他需要记录你要回滚的相应日志信息。
例如

(1)当你delete一条数据的时候，就需要记录这条数据的信息，回滚的时候，insert这条旧数据
(2)当你update一条数据的时候，就需要记录之前的旧值，回滚的时候，根据旧值执行update操作
(3)当年insert一条数据的时候，就需要这条记录的主键，回滚的时候，根据主键执行delete操作
undo log记录了这些回滚需要的信息，当事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。

ps:具体的undo log日志长啥样，这个可以写一篇文章了。而且写出来，看的人也不多，姑且先这么简单的理解吧。

# 问题三: Mysql怎么保证持久性的？
OK，是利用Innodb的redo log。
正如之前说的，Mysql是先把磁盘上的数据加载到内存中，在内存中对数据进行修改，再刷回磁盘上。如果此时突然宕机，内存中的数据就会丢失。
怎么解决这个问题？

简单啊，事务提交前直接把数据写入磁盘就行啊。
这么做有什么问题？

只修改一个页面里的一个字节，就要将整个页面刷入磁盘，太浪费资源了。毕竟一个页面16kb大小，你只改其中一点点东西，就要将16kb的内容刷入磁盘，听着也不合理。
毕竟一个事务里的SQL可能牵涉到多个数据页的修改，而这些数据页可能不是相邻的，也就是属于随机IO。显然操作随机IO，速度会比较慢。
于是，决定采用redo log解决上面的问题。当做数据修改的时候，不仅在内存中操作，还会在redo log中记录这次操作。当事务提交的时候，会将redo log日志进行刷盘(redo log一部分在内存中，一部分在磁盘上)。当数据库宕机重启的时候，会将redo log中的内容恢复到数据库中，再根据undo log和binlog内容决定回滚数据还是提交数据。

采用redo log的好处？
其实好处就是将redo log进行刷盘比对数据页刷盘效率高，具体表现如下

redo log体积小，毕竟只记录了哪一页修改了啥，因此体积小，刷盘快。
redo log是一直往末尾进行追加，属于顺序IO。效率显然比随机IO来的快。

# 问题四: Mysql怎么保证隔离性的？
利用的是锁和MVCC机制。还是拿转账例子来说明，有一个账户表如下
表名t_balance


id | user_id | balance
---|---|---
1 | A| 200
2 | B| 0


其中id是主键，user_id为账户名，balance为余额。还是以转账两次为例，如下图所示



[图片看不到点这个哈](https://files.jb51.net/file_images/article/201905/2019510145312535.jpg?2019410145319)


![image](https://files.jb51.net/file_images/article/201905/2019510145312535.jpg?2019410145319)

至于MVCC,即多版本并发控制(Multi Version Concurrency Control),一个行记录数据有多个版本对快照数据，这些快照数据在undo log中。
如果一个事务读取的行正在做DELELE或者UPDATE操作，读取操作不会等行上的锁释放，而是读取该行的快照版本。
由于MVCC机制在可重复读(Repeateable Read)和读已提交(Read Commited)的MVCC表现形式不同，就不赘述了。
但是有一点说明一下，在事务隔离级别为读已提交(Read Commited)时，一个事务能够读到另一个事务已经提交的数据，是不满足隔离性的。但是当事务隔离级别为可重复读(Repeateable Read)中，是满足隔离性的。

这里没有对锁和MVCC机制进行具体的描述，我觉得锁和MVCC可以再来一个章节了

