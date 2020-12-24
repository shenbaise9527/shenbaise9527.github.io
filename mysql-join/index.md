# MySQL的join分析


## 问题
1. 使用join时驱动表、被驱动表是如何选择的?影响因素有哪些?
2. 如何优化?

## 数据准备
``` sql
/*创建表*/
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

/*创建存储过程*/
delimiter ;;
create procedure idata()
begin
    declare i int;
    set i = 1;
    while(i<=1000) do
        insert into t2 values(i, i, i);
        set i=i+1;
    end while;
end;;
delimiter ;

/*执行*/
call idata();

/*创建表t1*/
create table t1 like t2;

/*插入数据*/
insert into t1 (select * from t2 where id <= 100);
```

## Index Nested-Loop Join(NLJ)
join时能用上被驱动表的索引,称之为`Index Nested-Loop Join`,简称为NLJ
``` sql
mysql> explain select * from t1 join t2 on t1.a=t2.a;
+----+-------------+-------+------------+------+---------------+------+---------+----------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref      | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+----------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL     |  100 |   100.00 | Using where |
|  1 | SIMPLE      | t2    | NULL       | ref  | a             | a    | 5       | web.t1.a |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+------+---------+----------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
从上面的结果可以看出,驱动表是表`t1`,被驱动表是表`t2`.驱动表是全表扫描,而被驱动表是用的索引`a`.

语句执行过程是:
1. 从表`t1`中读入一行数据R.
2. 从R取出字段`a`的值,去表`t2`里查找.
3. 取出表`t2`中满足条件的行,跟R组成一行,作为结果集的一部分.
4. 重复执行步骤1到3,直到表`t1`的末尾,循环结束.

假定驱动表有N行,被驱动表有M行,每扫描一行驱动表,使用字段a的值去被驱动表的索引树a上查找,然后再回表到被驱动表的主键索引树,则被驱动表要扫描2*$log_2{M}$.则总的扫描行数为N+N\*2\*$log_2{M}$.显然N对扫描行数的影响更大,因此在这种情况下应该使用小表为驱动表.

但如果被驱动表没有索引列?

## Block Nested-Loop Join(BNL)
join时被驱动表没有索引时,称之为`Block Nested-Loop Join`,简称为BNL
``` sql
/*使用straight_join强行指定驱动表为t1*/
mysql> explain select * from t1 straight_join t2 on t1.a=t2.b;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL |  100 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```
从上面的Extra字段的值`Using join buffer (Block Nested Loop)`可以看出,采用的正是BNL.

语句执行过程是:
1. 把表`t1`的数据读入线程内存`join_buffer`中
2. 扫描表`t2`,把每一行取出,跟`join_buffer`中的数据比对,满足`join`条件的,作为结果集的一部分.

整个过程表`t1`和表`t2`都是全表扫描,扫描行数为1000+100=1100,由于`join_buffer`中的数据时无序的,对表`t2`里的每一行都要做100次判断,总判断次数为1000*100=10万次.判读次数是纯内存操作,相比读表会快上不少,整个过程就是扫描1100行+10万次内存操作.

***在这种情况下,不论选择哪个表为驱动表其实是没有差异的.***

`join_buffer`的大小是由参数`join_buffer_size`来控制的,默认大小为256k.
``` sql
mysql> show variables like 'join%';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| join_buffer_size | 262144 |
+------------------+--------+
1 row in set (0.26 sec)
```

如果表`t1`的数据量很大,导致`join_buffer`放不下了,整个过程又会是怎么样的?

策略很简单,就是分段放入,分次比较.详见[官网说明](https://dev.mysql.com/doc/internals/en/join-buffer-size.html)
Basic information about the join buffer cache:
* The size of each join buffer is determined by the value of the join_buffer_size system variable.
* This buffer is **used only when the join is of type ALL or index (in other words, when no possible keys can be used)**.
* A join buffer is never allocated for the first non-const table, even if it would be of type ALL or index.
* The buffer is allocated when we need to do a full join between two tables, and freed after the query is done.
* Accepted row combinations of tables before the ALL/index are stored in the cache and are used to compare against each read row in the ALL table.
* We only store the used columns in the join buffer, not the whole rows.

Assume you have the following join:
``` sql
Table name Type
t1         range
t2         ref
t3         ALL
```
The Join is then done as follows:
``` sql
- While rows in t1 matching range
 - Read through all rows in t2 according to reference key
  - Store used fields from t1, t2 in cache
  - If cache is full 
    - Read through all rows in t3 
      - Compare t3 row against all t1, t2 combinations in cache 
        - If row satisfies join condition, send it to client 
    - Empty cache 

- Read through all rows in t3
 - Compare t3 row against all stored t1, t2 combinations in cache
   - If row satisfies join condition, send it to client
```

假设驱动表的行数是N,需要分K段才能完成算法流程,被驱动表数据行数是M.显然N越大,K就会越大,K=$\lambda$\*N,$\lambda$取值范围为(0,1).此算法扫描的行数为N+$\lambda$\*N\*M,内存判断次数为N*M,在N和M确定的情况下,N小些,扫描行数的算式会更小些,可见此时应小表当驱动表.参数$\lambda$才是影响扫描行数的关键因素,这个值应该越小越好.在N固定时若`join_buffer_size`越大,能放入的行数越多,该值就会越小.

若join语句很慢,可尝试把`join_buffer_size`改大点.

以上两个join算法都应该尽量使用小表作为驱动表,但什么是小表列?

## 什么是小表?
先来看看两组sql语句
``` sql
/*Q1,此时放入join_buffer的数据是100行*/
mysql> explain select * from t1 straight_join t2 on t1.b=t2.b where t2.id<=50;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |  100 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t2    | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   50 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)

/*Q2,此时放入join_buffer的数据是50行*/
mysql> explain select * from t2 straight_join t1 on t1.b=t2.b where t2.id<=50;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t2    | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   50 |   100.00 | Using where                                        |
|  1 | SIMPLE      | t1    | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |  100 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

/*Q3,此时放入join_buffer的数据是100行,但只需包含t1.b这一个字段*/
mysql> explain select t1.b,t2.* from t1 straight_join t2 on t1.b=t2.b where t2.id<=100;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |  100 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t2    | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |  100 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

/*Q4,此时放入join_buffer的数据是100行,但需包含表t2的所有字段*/
mysql> explain select t1.b,t2.* from t2 straight_join t1 on t1.b=t2.b where t2.id<=100;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t2    | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |  100 |   100.00 | Using where                                        |
|  1 | SIMPLE      | t1    | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |  100 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```
* 针对语句Q1和Q2来说,Q2放入`join_buffer`的数据只有50行,相对来说此时t2是小表,应该为驱动表.
* 针对语句Q3和Q4来说,放入`join_buffer`的行数是一样的,但Q3只需要放入表t1的一个字段,相对来说此时t1是小表,应该为驱动表.

**在决定哪个表为驱动表时,应该要两个表按照各自的条件过滤,然后计算参与join的各个字段的总数据量,数据量小的那个表就是小表,应该为驱动表.**

## 优化
### Multi-Range Read(MRR)优化
MRR优化的主要目的是使用顺序读盘.

在查询过程中使用非主键索引时需要进行回表操作去主键索引树中获取相关字段的值.在非主键索引树中获取的主键ID不是有序的,当循环回表时会触发主键索引树的磁盘随机读取,而这是最耗时的操作.

而MRR优化过程:
1. 从非主键索引中把符合的主键ID先全部读取出来,放入`read_rnd_buffer`中
2. 把`read_rnd_buffer`中的主键ID排序
3. 根据排序后的主键ID去主键索引树中查找记录

`read_rnd_buffer`是由参数`read_rnd_buffer_size`控制的,默认为256k
``` sql
mysql> show variables like 'read_rnd%';
+----------------------+--------+
| Variable_name        | Value  |
+----------------------+--------+
| read_rnd_buffer_size | 262144 |
+----------------------+--------+
1 row in set (0.01 sec)
```

当`read_rnd_buffer`满时,就会执行步骤2和3,然后清空`read_rnd_buffer`,之后继续找非主键索引的下个记录,并继续循环.

MRR能提升性能的核心在于,在非主键索引上是一个范围查询,可以有足够的主键ID,这样排序后,再去主键索引查找数据,才能体现出顺序性的优势.

若想稳定地使用MRR优化,需要设置`set optimizer_switch="mrr_cost_based=off"`(官方文档:优化器在判断消耗时,会更倾向不使用MRR,把mrr_cost_based设置为off,就是固定使用MRR).
``` sql
/*开启MRR*/
mysql> set optimizer_switch="mrr_cost_based=off";
Query OK, 0 rows affected (0.00 sec)

/*使用了MRR*/
mysql> explain select * from t2 where a >= 100 and a <= 200;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------+
|  1 | SIMPLE      | t2    | NULL       | range | a             | a    | 5       | NULL |  101 |   100.00 | Using index condition; Using MRR |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------+
1 row in set, 1 warning (0.04 sec)
```
***需要注意,当未使用MRR优化时,查询返回的记录是按照索引`a`来排序的,但使用了MRR优化时,返回的记录若`read_rnd_buffer`未满时是按照主键ID来排序的;若满记录就不是有序的了.***

### Batched Key Access(BKA)
BKA算法是基于MRR对NLJ算法做的优化.

从驱动表中取出数据放入到`join_buffer`中,然后按照被驱动表的索引键排序,顺序去被驱动表查找相关数据.相比于NLJ的过程,BKA使用了MRR优化的思路,在被驱动表中是顺序读取,避免随机读取.

要启用BKA,需要设置参数.
``` sql
/*启动BKA,要先启动MRR*/
mysql> set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
Query OK, 0 rows affected (0.00 sec)

/*NLJ已被优化为BKA*/
mysql> explain select * from t1 join t2 on t1.a=t2.a;
+----+-------------+-------+------------+------+---------------+------+---------+----------+------+----------+----------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref      | rows | filtered | Extra                                  |
+----+-------------+-------+------------+------+---------------+------+---------+----------+------+----------+----------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL     |  100 |   100.00 | Using where                            |
|  1 | SIMPLE      | t2    | NULL       | ref  | a             | a    | 5       | web.t1.a |    1 |   100.00 | Using join buffer (Batched Key Access) |
+----+-------------+-------+------------+------+---------------+------+---------+----------+------+----------+----------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

### BNL算法的性能问题
基于BNL算法时,由于`join_buffer`容量有限,如果驱动表是大表时需要进行分段处理,这样会导致被驱动表进行多次扫描,如果被驱动表是一个大的冷数据库,除了导致IO压力大外,还有什么其它影响?

从磁盘读取数据后是放入到`Buffer Pool`里的,而`Buffer Pool`的容量也是有限的,当空间不够时会淘汰一些老数据页来容纳新的数据页,淘汰算法InnoDB引擎使用的是变种LRU算法.

把`Buffer Pool`按照3:5的比例划分为old和young区域,从磁盘读取的数据先放入到old区域,如果超过1秒该数据页还在old区域且还有被访问就会被移入到young区域,在old和young区域都是LRU算法来淘汰老数据的.

若被驱动表的数据量小于old区域(即整个表能全部放入到old区域),由于需要多次扫描被驱动表,而时间间隔可能超过1秒,这会导致这部分数据会被移入到young区域.

若被驱动表的数据量超过了old区域,在遍历的过程中会涉及到淘汰old区域的数据来存放该表的数据,这样会导致业务正常访问的数据没有机会放入到young区域(没有超过1秒就被淘汰出old了).

以上两种情况都会对`Buffer Pool`的正常运作产生影响.

BNL算法对系统的影响主要包括:
* 可能会多次扫描被驱动表,占用磁盘IO.
* 判断join条件需要执行M*N次对比,如果大表就会占用CPU资源.
* 可能会导致`Buffer Pool`的热数据被淘汰,影响内存命中率.

优化策略:
* 在被驱动表建索引,把BNL转化为BKA.
* 若被驱动表不适合加索引,可使用临时表(`create temporary`),把被驱动表满足条件的记录放入临时表,给临时表加索引,仍然转化为BKA.
* hash join,但目前MySQL不支持,在`join_buffer`中支持hash,被驱动表的数据可以通过hash查找能快速定位,而不用再去执行M*N次比对了.

