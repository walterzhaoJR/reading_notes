# 3.事务隔离：为什么你改了我还看不见
* MySQL的事务是在引擎层面实现的，但并不是所有的引擎都支持事务
* 说到事务就会想到ACID。A：原子性，C：一致性，I：隔离性，D：持久性
* 数据库上如果有多个事务执行就会产生：脏读、不可重复读、幻度（phantom read），为了解决这些问题就有了隔离级别的概念
* SQL标准的事务隔离级别：
  * read uncommitted：一个事务没有提交，他的改变就可以被其他的事务看到
  * read committed：一个事务提交后，他的改变，其他的事务才可以看到
  * repeatable read：一个事务执行过程中看到的数据，总是个这个事务启动时看到的数据一致，当然在rr下，未提交的变更度其他事务也是不可见的
  * serializable：串行化，对同一行记录，读会加读锁，写会加写锁，事务出现冲突的时候，后访问的事务，要先等先访问的事务结束，才能访问
* 实际中数据库会创建一个视图：访问的时候一视图的逻辑结果为准。
  * 在可重复读的隔离级别下：视图是在事务启动的时候创建
  * 在读以提交的时候：视图是在每条sql开始执行的时候创建
  * 读为提交：没有使用视图的概念
  * 可串行化：使用锁来避免并行访问
* 配置隔离级别：修改transaction-isolation
* 隔离界别的实现（RR为例）：
  * 每条记录在更新的时候都会记录回滚操作
  * 不同时间启动的事务会有不同的read-view。同一个记录在数据库中有多个版本，这就是MVCC（多版本并发控制）
  * 回滚日志不会一直保留：系统在判断当前没有事务使用的回滚日志就会被删掉（系统里没有比这个回滚日志更早的事务）
  * 尽量不要使用长事务：在这个事务提交前，数据库里面可能用的到的回滚记录都要保存，这就会到时占用很大空间。
* 事务的启动方式：
  * begin或start transaction开启，commit提交，rollback回滚
  * set autocommit=0，这就意味只要执行select就会开启事务，这个事务持续存在，不会自动提交，直到commit或rollback
  * 如果对于一个需要频繁使用事务的业务，可以设置autocommit为0，并且使用commit work and chain语法，提交之前的事务并且开启新的事务
* 可以在information_schema库中的innodb_trx的表中查找长事务（超过60秒）：

```sql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started)) > 60;
```

* 注意：5.7引入了transaction_isolation用来替换老版本的tx_transaction，8.0.3去掉了后者

* undo日志也会记录到redo中，如果一个事务执行到一半实例崩溃，恢复的时候先恢复redo，再根据redo构造undo回滚到宕机前没有提交的事务

* 几种异常：

  * 脏读：
    当数据库中一个事务A正在修改一个数据但是还未提交或者回滚，
    另一个事务B 来读取了修改后的内容并且使用了，
    之后事务A提交了，此时就引起了脏读。 此情况仅会发生在： 读未提交的的隔离级别.
  * 不可重复读：
    在一个事务A中多次操作数据，在事务操作过程中(未最终提交)，
    事务B也才做了处理，并且该值发生了改变，这时候就会导致A在事务操作
    的时候，发现数据与第一次不一样了。 就是不可重复读。此情况仅会发生在：读未提交、读提交的隔离级别.

  * 幻读：
    一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为幻读。幻读是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，比如这种修改涉及到表中的“全部数据行”。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入“一行新数据”。那么，以后就会发生操作第一个事务的用户发现表中还存在没有修改的数据行，就好象发生了幻觉一样.
    一般解决幻读的方法是增加范围锁RangeS，锁定检索范围为只读，这样就避免了幻读。此情况会回发生在：读未提交、读提交、可重复读的隔离级别.