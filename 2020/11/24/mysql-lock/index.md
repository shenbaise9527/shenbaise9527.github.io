# MySQL锁机制分析


## 全局锁
全局锁就是对整个数据库实例加锁.MySQL提供了命令`Flush tables with read lock`(FTWRL),可使整库处于只读状态,其它线程的数据更新语句(数据的增删改)、数据定义语句(DDL)和更新类事务的提交语句都会被阻塞.

全局锁的典型使用场景是做全库的逻辑备份.使用FTWRL可确保不会有其它线程对数据库做更新,然后再对整个库做备份,这样可以保证数据库的数据逻辑一致性.但让整库只读,是危险操作:
1. 如果在主库上备份,那么在备份期间都不能执行更新,业务基本上停摆.
2. 如果在从库上备份,那么在备份期间不能执行主库同步过来的binlog,会主从延迟.

在针对InnoDB引擎的表做全库备份时,可以采用可重复读隔离级别下来进行备份,不会对其它线程的操作造成堵塞,还可以保证数据的逻辑一致性.是基于一致性视图+MVCC来实现的.

可以使用官方工具mysqldump,使用参数`-single-transaction`时,会在可重复读隔离级别下启动一个事务,确保拿到一致性视图.但该方法只适用于所有的表都使用事务引擎的库.

使用命令`set global readonly=true`也可使整库处于只读状态,但这个操作更加的危险.
1. 在有些系统中,readonly的值会被用来做其它逻辑,比如判断一个库是主库还是备库.修改该值影响面更广.
2. 在异常处理机制上有差异.执行FTWRL后客户端异常断开,MySQL会自动释放这个全局锁,使数据库恢复到正常状态.而readonly修改后会一直有效,使库一直处于只读状态,风险更高.

ps: 在从库上如果用户有超级权限,readonly是失效的.

## 表级锁
MySQL里表级别的锁有两种,表锁和元数据锁(meta data lock,简称MDL锁)

### 表锁
加锁语法为`lock tables ... read/write`,解锁语句为`unlock tables`
``` sql
-- session1
mysql> lock tables t read, t1 write;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t limit 1;
+----+--------+------+-----+------+
| id | city   | name | age | addr |
+----+--------+------+-----+------+
|  1 | gongan | ??   |  80 | test |
+----+--------+------+-----+------+
1 row in set (0.00 sec)

mysql> select * from t1 limit 1;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
+----+------+------+
1 row in set (0.00 sec)

mysql> insert into t values (null, 'gongan', 'guanguan', 1, 'test');
ERROR 1099 (HY000): Table 't' was locked with a READ lock and can't be updated
mysql> insert into t1 values(101, 101, 101);
Query OK, 1 row affected (0.00 sec)

mysql> select * from t2 limit 1;
ERROR 1100 (HY000): Table 't2' was not locked with LOCK TABLES
mysql> insert into t2 values(1001, 1001, 1001);
ERROR 1100 (HY000): Table 't2' was not locked with LOCK TABLES
```
* 针对表`t`读锁,表`t1`为写锁.本线程两个表的查询操作都正常,表`t1`插入正常,但针对表`t`的插入操作报错.
* 本线程只能操作表`t`和表`t1`,操作其它表都会报错.
* 本线程只能读表`t`,写会报错.
* 本线程能读写表`t1`.

``` sql
-- session2
mysql> select * from t limit 1;
+----+--------+------+-----+------+
| id | city   | name | age | addr |
+----+--------+------+-----+------+
|  1 | gongan | ??   |  80 | test |
+----+--------+------+-----+------+
1 row in set (0.00 sec)

mysql> select * from t2 limit 1;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
+----+------+------+
1 row in set (0.00 sec)

mysql> insert into t2 values(1001, 1001, 1001);
Query OK, 1 row affected (0.03 sec)

-- block
mysql> insert into t values (null, 'gongan', 'guanguan', 1, 'test');
```
* 其它线程能正常读表`t`,但写表`t`时被阻塞
* 其它线程能正常读写其它表.

``` sql
-- session3(block)
mysql> select * from t1 limit 1;
```
* 其它线程读表`t1`时被阻塞.

``` sql
-- session4(block)
mysql> insert into t1 values(102, 102, 102);
```
* 其它线程写表`t1`时被阻塞.

### 元数据锁
MySQL5.5版本之后的功能,会自动加锁、解锁.主要是为了解决DDL和DML并发的问题.DML时会加MDL读锁,DDL时会加MDL写锁,读读之间不互斥,读写、写写之间互斥.
``` sql
-- session1
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1 limit 1;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
+----+------+------+
1 row in set (0.00 sec)

-- session2
mysql> alter table t1 add f int;

-- session3
mysql> select * from t1 limit 1;
```
1. 在session1提交事务之前,session2和session3都被阻塞了.
2. session1是获取了表`t1`的MDL读锁,但由于事务未提交,导致该MDL读锁未被释放.
3. session2是给表`t1`增加列,需要MDL写锁,由于读写互斥,导致该操作被堵塞.
4. session3是读表`t1`,需要MDL读锁,但之前已经有MDL写锁在等待了,也导致获取不到MDL读锁,从而被堵塞.

``` sql
-- session1
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

-- session2
mysql> alter table t1 add f int;
Query OK, 0 rows affected (4 min 23.27 sec)
Records: 0  Duplicates: 0  Warnings: 0

-- session3
mysql> select * from t1 limit 1;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
+----+------+------+
1 row in set (4 min 20.25 sec)
```
session1提交之后,session2和session3才执行完,但注意session3在session2之前执行,也就是session3先获取到了MDL读锁.

**当有多个线程在等待MDL锁时,获取锁的规则是什么?会由哪个线程得到锁?读锁优先还是写锁优先?**

***MDL锁是在需要的时候由MySQL自动加的,但要等待事务被提交后才会被释放.***

## 行锁
全局锁和表级锁是在server层实现的,而行锁是由引擎层实现的,这里主要关注InnoDB的行锁.行锁是自动加的.

### 两阶段协议锁
``` sql
-- session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t2 set a = 2 where id = 1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> update t2 set a = 3 where id = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

-- session B(block)
mysql> update t2 set a = 1 where id = 1;
```
可以看到session A未提交,会导致session B阻塞.当session A在`commit`提交之后,session B才能开始执行.

行锁是在需要的时候由InnoDB自动加上,但直到事务结束时锁才会被释放.这就是两阶段协议锁.

### 死锁和死锁检测
在并发系统中当不同的线程出现循环资源依赖,就会导致这些线程都进入无限等待的状态,称为死锁.如下session A和session B就出现了循环资源依赖,导致死锁.
|session A|session B|
|:--|:--|
|begin;update t2 set a = 2 where id = 1;|begin;|
||update t2 set a = 3 where id = 2;|
|update t2 set a = 4 where id = 2;||
||update t2 set a = 5 where id = 1;|

在InnoDB中,当处于锁等待状态时,就有可能会触发死锁检测,是参数`innodb_deadlock_detect`控制的.
``` sql
-- 默认值为on,表示开启死锁检测.
mysql> show variables like 'innodb_deadlock%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_deadlock_detect | ON    |
+------------------------+-------+
1 row in set (0.00 sec)
```

每个新来的被堵住的线程,都要判断会不会由于自己的加入导致了死锁,这是一个时间复杂度为O(n)的操作.假设有1000个线程要同时更新同一行,那么死锁检测操作就是100万量级的,这期间会消耗大量的CPU资源.注意死锁检测只会检测相关联的线程.比如当前session A在等待session B,而session B在等待session C;session D在等待session E.当session F加入需要等待session A时,只会检测F->A->B->C,D和E是不会检测的.

***怎么解决由热点行更新导致的性能问题?***

***若在事务中需要锁多个行,把最可能造成锁冲突、最可能影响并发度的锁的申请时机尽量往后放.***

另外需要注意的是,当等待锁一定时间后,会出现超时现象,是参数`innodb_lock_wait_timeout`控制的.
``` sql
-- 等待锁超时.
mysql> update t2 set a = 1 where id = 1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

-- 锁等待超时时间,默认为50s.
mysql> show variables like 'innodb_lock_wait_timeout';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_lock_wait_timeout | 50    |
+--------------------------+-------+
1 row in set (0.01 sec)
```

死锁时可以通过`show engine innodb status`命令查看
``` sql
mysql> show engine innodb status\G
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2020-07-14 14:49:16 0x7fa8507f8700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 26 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 521 srv_active, 0 srv_shutdown, 3098384 srv_idle
srv_master_thread log flush and writes: 3098905
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 6262
OS WAIT ARRAY INFO: signal count 28687
RW-shared spins 0, rounds 27977, OS waits 2270
RW-excl spins 0, rounds 159415, OS waits 2170
RW-sx spins 4327, rounds 34223, OS waits 205
Spin rounds per wait: 27977.00 RW-shared, 159415.00 RW-excl, 7.91 RW-sx
-- 死锁信息
------------------------
LATEST DETECTED DEADLOCK
------------------------
2020-07-14 12:00:34 0x7fa8507f8700
*** (1) TRANSACTION:
TRANSACTION 3667, ACTIVE 7 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 73, OS thread handle 140361007904512, query id 1776328 localhost web updating
update t set d=d+1 where c=10
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 43 page no 4 n bits 80 index c of table `web`.`t` trx id 3667 lock_mode X waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 3668, ACTIVE 12 sec inserting
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
MySQL thread id 72, OS thread handle 140360881768192, query id 1776329 localhost web update
insert into t values(8, 8, 8)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 43 page no 4 n bits 80 index c of table `web`.`t` trx id 3668 lock mode S
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 43 page no 4 n bits 80 index c of table `web`.`t` trx id 3668 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;

*** WE ROLL BACK TRANSACTION (1)
```

### 锁等待分析
``` sql
-- session A,开启事务,在di=1加行锁.
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t2 set a = 2 where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

-- session B(block),等待session A的锁.
mysql> select * from t2 where id = 1 lock in share mode;

-- session C,查看阻塞情况.
mysql> show processlist;
+----+------+-----------+------+---------+------+------------+--------------------------------------------------+
| Id | User | Host      | db   | Command | Time | State      | Info                                             |
+----+------+-----------+------+---------+------+------------+--------------------------------------------------+
| 68 | web  | localhost | web  | Sleep   |  104 |            | NULL                                             |
| 69 | web  | localhost | web  | Query   |   20 | statistics | select * from t2 where id = 1 lock in share mode |
| 70 | web  | localhost | web  | Query   |    0 | starting   | show processlist                                 |
| 71 | web  | localhost | web  | Sleep   | 3724 |            | NULL                                             |
+----+------+-----------+------+---------+------+------------+--------------------------------------------------+
4 rows in set (0.00 sec)

-- 从下面可以看出69在等待68的锁.
mysql> select * from sys.innodb_lock_waits where locked_table='`web`.`t2`'\G
*************************** 1. row ***************************
                wait_started: 2020-07-13 16:37:31
                    wait_age: 00:00:02
               wait_age_secs: 2
                locked_table: `web`.`t2`
                locked_index: PRIMARY
                 locked_type: RECORD
              waiting_trx_id: 421836271774456
         waiting_trx_started: 2020-07-13 16:37:31
             waiting_trx_age: 00:00:02
     waiting_trx_rows_locked: 1
   waiting_trx_rows_modified: 0
                 waiting_pid: 69
               waiting_query: select * from t2 where id = 1 lock in share mode
             waiting_lock_id: 421836271774456:33:5:2
           waiting_lock_mode: S
             blocking_trx_id: 3553
                blocking_pid: 68
              blocking_query: NULL
            blocking_lock_id: 3553:33:5:2
          blocking_lock_mode: X
        blocking_trx_started: 2020-07-13 16:36:07
            blocking_trx_age: 00:01:26
    blocking_trx_rows_locked: 1
  blocking_trx_rows_modified: 1
     sql_kill_blocking_query: KILL QUERY 68
sql_kill_blocking_connection: KILL 68
1 row in set, 3 warnings (0.06 sec)
```
在MySQL5.7及之后版本可以通过`sys.innodb_lock_waits`表获取锁占用情况.

### 幻读
#### 什么是幻读?
1. 在一个事务内,前后看到的数据不一致,不一致特指后面看到的数据行数多了,这就是幻读(幻读特指新插入的行).
2. 在读已提交隔离级别下,是允许存在幻读现象的.
3. 在可重复读隔离级别下,MySQL是不存在幻读现象的.
4. 在可重复读隔离级别下,普通的查询都是快照读,是不能看到别的事务插入的数据,因此必须是针对当前读的语句.

#### 幻读有什么问题?
在可重复隔离级别下,如果允许幻读会出现什么现象?
``` sql
-- 隔离级别为可重复读.
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
1 row in set (0.00 sec)

-- session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t2 set b = a + 1 where a = 4;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

-- session B,假定此处不会block
mysql> insert into t2 values(1002, 4, 1002);

-- session A,提交事务
mysql> commit;
Query OK, 0 rows affected (0.01 sec)
```
如上所示的流程,假如session A只是对a=4这行加了行锁.

session B的事务是先提交,session A的事务是后提交.此时数据库的数据为(4, 4, 5)和(1002, 4, 1002).但在binlog中先是session B的操作,然后是session A的操作,该binlog同步到从库上开始执行,会得到什么结果?

session B先执行,插入一行数据(1002, 4, 1002),然后更新a=4的行,从库的数据为(4, 4, 5)和(1002, 4, 5),此时主库和从库的数据不一致了.

session A本来是想更新所有a=4的行的,但在这之后session B插入了一行a=4的数据,导致session A的语义被破坏了.

***主从数据不一致和语义被破坏,这就是幻读的问题.***

实际上InnoDB在可重复读级别下是不会出现幻读的现象的,上面的sql语句,session B的`insert`操作会被阻塞,直到session A的事务提交后才能执行.

#### 如何解决幻读?
通过上例,只是加行锁,无法阻止幻读的出现.InnoDB是通过加间隙锁(Gap Lock),来锁住a=4的间隙,这样可以阻塞别的线程的插入操作.

间隙锁,锁的就是两个值之间的间隙,以上例session A来说,会锁住(3, 4)和(4, 5)的间隙.这样再插入session B的数据,会落入到(4, 5)间隙,导致被阻塞.

间隙锁一般是针对可重复隔离级别的.读已提交一般情况下只有行锁(说明该隔离级别会出现幻读的现象).

间隙锁的引入会导致锁的范围变大,这样其实会影响并发度的.

***需要注意,间隙锁之间是不互斥的,不同的session之间可以锁住同样的间隙***

#### Next-key lock
行锁+间隙锁合称为next-key lock,每个next-key lock都是前开后闭区间.

### InnoDB加锁规则(可重复读隔离级别)
1. 原则一: 加锁的基本单位是next-key lock,是前开后闭区间.
2. 原则二: 查找过程中访问到的对象才会加锁.
3. 优化一: 索引上的等值查询,给唯一索引加锁的时候,next-key lock会退化为行锁.
4. 优化二: 索引上的等值查询,向右遍历时且最后一个值不满足等值条件的时候,next-key lock会退化为间隙锁.
5. bug一: 唯一索引上的范围查询会访问到不满足条件的第一个值为止.

数据准备.
``` sql
-- 创建表t.
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB

-- 插入数据.
mysql> insert  into t values(0, 0, 0),(5, 5, 5),(10, 10, 10),(15, 15, 15),(20, 20, 20),(25, 25, 25);
Query OK, 6 rows affected (0.00 sec)
Records: 6  Duplicates: 0  Warnings: 0
```

#### 等值查询间隙锁
``` sql
-- session A.
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t set d=d+1 where id=7;
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0

-- session B(block).
mysql> insert into t values(8,8,8);

-- session C.
mysql> update t set d=d+1 where id=10;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
1. session A在主键索引上加锁,根据规则一加锁为(5, 10],根据优化二退化为间隙锁(5, 10)
2. session B要插入id=8,落在间隙锁(5, 10)之间,被阻塞.
3. session C更新id=10的行,根据规则一加锁为(5, 10],根据优化一退化为行锁(10),锁不冲突,可以更新成功.

#### 非唯一索引等值锁
``` sql
-- session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select id from t where c = 5 lock in share mode;
+----+
| id |
+----+
|  5 |
+----+
1 row in set (0.00 sec)

-- session B
mysql> update t set d = d+1 where id = 5;
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0

-- session C(block)
mysql> insert into t values(7, 7, 7);
```
1. session A在索引c上加锁,根据规则一为(0, 5]和(5, 10],根据优化二退化为(0, 5]和(5, 10).注意该语句是使用了覆盖索引,所以并没有在主键索引上加锁.
2. session B是在主键索引上加锁,根据优化一退化为行锁(5),和session A并不冲突,可以更新成功.
3. session C要再索引c上插入c=7的行,落在了(5, 10)之间,被session A阻塞.

**注意:语句中使用的是`lock in share mode`,是读锁且使用了覆盖索引,并不需要访问主键索引.但如果使用的是`for update`,又是什么效果列?**

#### 主键索引范围锁
``` sql
--session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where id >= 10 and id < 11 for update;
+----+------+------+
| id | c    | d    |
+----+------+------+
| 10 |   10 |   10 |
+----+------+------+
1 row in set (0.01 sec)

--session B
mysql> insert into t values (8, 8, 8);
Query OK, 1 row affected (0.00 sec)

-- block
mysql> insert into t values(13, 13, 13);

--session C (block)
mysql> update t set d=d+1 where id=15;
```
1. session A在主键索引上加锁,满足条件的第一行是id=10,则加锁(5, 10],但根据优化一退化为行锁(10);范围查询要继续查找,则加锁(10, 15],由于是范围查询没有适用的优化规则.故加锁范围为[10, 15]
2. session B第一条插入id=8,不在锁的范围,可以插入成功.第二条插入id=13,在锁的范围,被阻塞.
3. session C更新id=15的行,在锁的范围,被阻塞.

#### 非唯一索引范围锁
``` sql
-- session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where c>=10 and c<11 for update;
+----+------+------+
| id | c    | d    |
+----+------+------+
| 10 |   10 |   10 |
+----+------+------+
1 row in set (0.00 sec)

-- session B(block)
mysql> insert into t values (8, 8, 8);

-- session C(block)
mysql> update t set d=d+1 where c=15;
```
1. session A在索引c上加锁,满足条件的第一行是c=10,则加锁(5, 10],注意c是非唯一索引没有优化规则;范围查询要继续查找,则加锁(10, 15],由于是范围查询没有适用的优化规则.故加锁范围为(5, 15].
2. session B要插入c=8的行,在锁的范围内,被阻塞.
3. session C要更新c=15的行,在锁的范围内,被阻塞.

#### 唯一索引范围bug
``` sql
--session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where id>10 and id<=15 for update;
+----+------+------+
| id | c    | d    |
+----+------+------+
| 15 |   15 |   15 |
+----+------+------+
1 row in set (0.00 sec)

--session B(block)
mysql> update t set d=d+1 where id=20;

--session C(block)
mysql> insert into t values(16, 16, 16);
```
1. session A在主键索引上加锁,按照原则一,加锁(10, 15],但InnoDB会扫描到第一个不满足条件的行为止,这里也就是id=20，由于是范围扫描,所以还会加锁(15, 20].但(15, 20]是完全没必要的,可以认为是bug.
2. session B要更新id=20的行,在锁的范围内,被阻塞.
3. session C要插入id=16的行,在锁的范围内,被阻塞.

#### 非唯一索引上存在相同键
``` sql
-- 插入新行,c=10的有两行数据.
mysql> insert into t values(30, 10, 30);
Query OK, 1 row affected (0.00 sec)

-- 可以看到索引c的顺序.
mysql> select * from t order by c;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  0 |    0 |    0 |
|  5 |    5 |    5 |
| 10 |   10 |   10 |
| 30 |   10 |   30 |
| 15 |   15 |   15 |
| 20 |   20 |   20 |
| 25 |   25 |   25 |
+----+------+------+
7 rows in set (0.01 sec)

--session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from t where c=10;
Query OK, 2 rows affected (0.00 sec)

--session B(block)
mysql> insert into t values(12, 12, 12);

--session C
mysql> update t set d=d+1 where c=15;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
1. session A是在索引c上加锁,满足条件的第一行为(id=10,c=10),则加锁((id=5,c=5),(id=10,c=10)],非唯一索引没有优化规则.第一行为(id=30,c=10),则加锁((id=10,c=10),(id=30,c=10)],继续查找到(id=15,c=15)结束,根据优化规则二,退化为间隙锁((id=30,c=10),(id=15,c=15))
2. session B插入c=12的行,在锁的范围内,被阻塞.
3. session C更新c=15的行,不再锁的范围内,可正常更新.

#### limit语句加锁
``` sql
--session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from t where c=10 limit 2;
Query OK, 2 rows affected (0.00 sec)

--session B(正常)
mysql> insert into t values(12, 12, 12);
Query OK, 1 row affected (0.02 sec)
```
session A是在索引c上加锁,由于有`limit 2`在读取到满足条件的两行数据后就不会再继续查找,所以加锁范围相比上节的例子会变小,加锁范围为((id=5,c=5),(id=10,c=10)]和((id=10,c=10),(id=30,c=10)]
session B要插入c=12的行,不在加锁范围内,可正常插入.

**在不改变语义的前提下,删除数据的时候尽量加limit,可有效降低加锁范围**

#### 死锁
``` sql
--session A
mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  0 |    0 |    0 |
|  5 |    5 |    5 |
| 10 |   10 |   10 |
| 15 |   15 |   15 |
| 20 |   20 |   20 |
| 25 |   25 |   25 |
+----+------+------+
6 rows in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select id from t where c=10 lock in share mode;
+----+
| id |
+----+
| 10 |
+----+
2 rows in set (0.00 sec)

--session B(block)
mysql> update t set d=d+1 where c=10;

--session A
mysql> insert into t values(8, 8, 8);
Query OK, 1 row affected (0.12 sec)

--session B(死锁,回滚)
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
1. 首先session A的加锁范围为next-key lock(5, 10]和间隙锁(10, 15)
2. session B的update语句被阻塞,其加锁逻辑为:先加间隙锁(5, 10),然后是行锁(10).由于间隙锁之间是不冲突的,有冲突的是行锁(10),导致sesssion B的阻塞.
3. 此时session A拥有(5, 10]和(10, 15)的锁,session B拥有(5, 10)的锁且在等待行锁(10).
4. 最后session A插入c=8的新行,落在间隙(5, 10)中,该间隙锁session B也拥有导致阻塞.
5. 此时会进行死锁检测,发现session B在等待session A的行锁,而session A又在等待session B的间隙锁,导致死锁.

#### order by加锁
``` sql
-- session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where c>=15 and c<=20 order by c desc lock in share mode;
+----+------+------+
| id | c    | d    |
+----+------+------+
| 20 |   20 |   20 |
| 15 |   15 |   15 |
+----+------+------+
2 rows in set (0.00 sec)

-- session B(block)
mysql> insert into t values(6, 6, 6);
```
1. session A由于是倒序,第一个访问的是索引c上最右边的c=20的行,会加间隙锁(20, 25)和next-key lock(15, 20],继续遍历到c=10才会停下来,则next-key lock(5, 10].故整个锁的范围是索引c上的(5, 25),主键索引上id=15和id=20的两个行锁.
2. session B要插入c=6的行,落在了间隙锁(5, 25)中,所以被阻塞.


### InnoDB加锁规则(读已提交隔离级别)
1. 主要是行锁(只有在外键场景下会有间隙锁)
2. 在语句执行过程中加的行锁,在语句执行完成后,就会把不满足条件的行的行锁释放掉,不需要等待事务提交.

在读已提交下锁的范围更小,锁的时间更短.

