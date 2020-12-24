# MySQL索引(InnoDB引擎)


## B+树
基于N叉树(每个父节点有N个子节点,子节点的值从左到右按照从小到大的顺序排列),非叶子节点只存储索引值,叶子节点储存索引值和数据,所有叶子节点采用链表串起来.

InnoDB采用的就是B+树,表的数据都是以索引的形式存放的,称为索引组织表.针对每个InnoDB引擎的表,都必须存在索引,当建表时没有显示声明索引,InnoDB会默认创建,如下`表t1`.
``` sql
/*mysql版本*/
Server version: 5.7.27-log MySQL Community Server (GPL)

/*创建表t,没有显示声明索引*/
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL
) ENGINE=InnoDB;
```

由于数据全部存储在叶子节点上,则每次查询都必须到叶子节点,其查找时间复杂度稳定,只和树高有关.
假如树高为4,N为1200,则这棵B+树能存储17亿多的数据量,整棵树的高度能维持在非常小的范围内,同时在内存里缓存前面若干层的节点,则极大的降低访问磁盘的次数,提高查询效率.

B+树的树高是由数据页大小和索引列的大小来决定的,而数据页大小是由参数`innodb_page_size`(默认为16k)的值决定;比如以一个`bigint`类型的字段为主键,则索引的大小为8字节(`bigint`大小)+6字节(指针大小,mysql里指针占6字节).则N的大小为16k/14字节,大约为1170.索引字段越小一个数据页能存放的数据就越多,N就会越大.

## 索引分类
### 主键索引
也被称为聚簇索引(clustered index),每个表都必须有且仅有一个主键索引(当没有显示声明主键索引时,InnoDB会默认创建一个以rowid为主键的索引),索引对应的字段的值唯一且不允许有空值.

主键索引的B+树的叶子节点的数据页里存放的是整行数据,且只有在主键索引的B+树中才有完整的数据.

### 非主键索引
又叫二级索引(secondary index),可以有多个,可分为唯一索引(索引对应的字段的值必须是唯一的,但允许为空值)和普通索引(对字段没什么限制,可重复可为空),对应的B+树的叶子节点的数据页里存放的是对应的主键值.

需要注意:当使用二级索引查询时只能获取到主键值和索引所对应列的值,要获取其它字段的值就只能根据主键值再去主键索引中查找,这个操作称为回表.
``` sql
/*mysql版本*/
Server version: 5.7.27-log MySQL Community Server (GPL)

/*创建表t*/
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128),
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB

/*根据普通索引city来查询,此时需要进行回表操作,来获取其它字段的值*/
mysql> select * from t1 where city='hangzhou';

/*根据主键索引来查询,直接在主键索引B+树上查找*/
mysql> select * from t1 where id=1;
```

## 索引维护
索引必然是有序的.当做增删改操作时,必须要进行必要的维护操作,来保持索引的有序性.在InnoDB中,读写都是以数据页为单位的(默认为16k),数据都是存放在数据页里的,当插入新值时为了保持索引的有序性,可能要把新值插入到数据页的中间位置,但如果此时数据页已满时会怎么样?

页分裂:当数据页已满时,引擎会申请一个新的数据页,然后挪动原数据页的一部分数据到新的数据页中.在这种情况下,插入性能是会受到影响的.另外数据页的空间利用率也会降低大约50%.

同理在删除数据时,有可能引发页合并(有分裂就有合并,是分裂的逆过程).

注意:
- 当新数据要插入到某个数据页的首位置,而此时该数据页已满,为了避免页分裂,会优先去找前一个数据页是否还有空余,若有就把新数据插入到前一个数据页的末尾位置.
- 当新数据要插入到某个数据页的尾位置,而此时该数据页已满,为了避免页分裂,会优先去找后一个数据页是否还有空余,若有就把新数据插入到后一个数据页的首位置.

为什么一般建议采用自增ID为主键?
* 自增ID自带有序性,每次插入都是追加操作,不涉及到挪动其它记录,也不会触发叶子节点的分裂.
* 每个二级索引的叶子节点上都是主键的值,而自增ID若是整型(int)为4字节,若是长整型(bigint)为8字节,主键长度越小,二级索引的叶子节点就越小,占用的空间也就越小.

## 索引优化
### 覆盖索引
为什么需要回表,是因为要去主键索引树中获取相应字段的值.但如果想要查询的字段是索引列的一部分列?
``` sql
/*仍以表t为例,先新加个索引,包含city和name字段*/
mysql> alter table t add index city_name(city, name);

/*通过city来查询name*/
mysql> explain select name from t where city = 'hangzhou';
+----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys  | key       | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city,city_name | city_name | 50      | const | 5505 |   100.00 | Using index |
+----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

/*通过city来查询name和age*/
mysql> explain select name, age from t where city = 'hangzhou';
+----+-------------+-------+------------+------+----------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys  | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+----------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | ref  | city,city_name | city | 50      | const | 5505 |   100.00 | NULL  |
+----+-------------+-------+------------+------+----------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
第一个查询采用的是索引city_name,Extra为`Using index`,表示是覆盖索引,需要查询的字段已包含在索引列中,不需要回表.

第二个查询采用的是索引city,Extra为`NULL`,则需要进行回表,去主键索引中获取相应字段的值.

覆盖索引能减少回表次数,即减少树的搜索次数,可显著提升性能.

### 前缀索引
- 联合索引的最左N个字段.
- 字符串索引的最左M个字符.

联合索引的值是按照索引定义里出现的字段顺序来排列的,比如表`t`的索引`city_name`是按照字段`city`和`name`的顺序排列的.
还是以表`t`为例,创建由字段`city`、`name`和`age`组成的联合索引.
``` sql
/*创建索引*/
mysql> alter table t add index city_name_age(city, name, age);

/*根据字段city和name来查询,使用了索引city_name_age*/
mysql> explain select * from t where city = 'hangzhou' and name = 'zhou';
+----+-------------+-------+------------+------+---------------+---------------+---------+-------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key           | key_len | ref         | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+---------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | ref  | city_name_age | city_name_age | 100     | const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+---------------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

联合索引的字段顺序如何安排?
- 如果通过调整顺序,可以少维护一个索引,则这个顺序是需要优先考虑的.
- 索引空间的大小.比如表t对`name`和`age`字段添加索引,基于空间考虑,应该增加一个(name,age)的联合索引和一个(age)的单索引,这时因为字段name比字段age大.

字符串索引,适用于模糊查询.
``` sql
/*在字符串字段city上新建索引*/
mysql> alter table t add index city(city);

/*模糊查询*/
mysql> explain select * from t where city like 'h%';
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | city          | city | 50      | NULL | 5505 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
可以看出`like 'h%'`模糊查询使用了索引`city`来快速查找.但如果是针对`like '%h%'`之类的模糊查询列?
``` sql
/*使用前后都模糊匹配模式,查询所有字段*/
mysql> explain select * from t where city like '%h%';
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 43868 |    11.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

/*使用前后都模糊匹配模式,查询id字段*/
mysql> explain select id from t where city like '%h%';
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | index | NULL          | city | 50      | NULL | 43868 |    11.11 | Using where; Using index |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
针对查询`select * from t where city like '%h%'`,使用的是全表扫描.

针对查询`select id from t where city like '%h%'`,使用的索引`city`,看rows字段也是全索引扫描,由于只需要获取字段`id`,故优化器认为顺序扫描索引`city`比主键索引的代价小.

***针对索引,一种是通过索引来快速查找,而另一种是通过索引来顺序遍历.***

**注意:**针对字段`city`和`age`的联合索引,也是适用于`city like 'h%'`的模糊查询.
``` sql
/*在字段city和age上新建联合索引*/
mysql> alter table t add index city_age(city, age);

/*利用联合索引的最左字段city的最左M个字符*/
mysql> explain select * from t where city like 'hang%';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | city_age      | city_age | 50      | NULL | 5505 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

联合索引(非主键索引)中的字段和主键索引所包含的字段有重复时,在非主键索引树中该字段不会出现多次.
``` sql
/*新建表geek*/
CREATE TABLE `geek` (
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  `d` int(11) NOT NULL,
  PRIMARY KEY (`a`,`b`),
  KEY `ca` (`c`,`a`),
  KEY `cb` (`c`,`b`)
) ENGINE=InnoDB
```
如上`geek`表中有3个索引,字段`a`和`b`组成的主键索引、字段`c`和`a`组成的普通索引、字段`c`和`b`组成的普通索引.
* 在`ca`索引树中,叶子节点包含的字段顺序为`c`、`a`和`b`.并不是`c`、`a`、`a`和`b`.
* 在`cb`索引树中,叶子节点包含的字段顺序为`c`、`b`和`a`.并不是`c`、`b`、`a`和`b`.

### 索引下推
针对前缀索引中的联合索引(`city`和`age`),适用于`city`的模糊查询,那如果查询条件再加上`age`会如何?
``` sql
/*在city字段上模糊匹配,在age上精确匹配*/
mysql> explain select * from t where city like 'hang%' and age = 10;
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | city_age      | city_age | 54      | NULL | 5505 |    10.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
可以看到仍然会使用索引`city_age`,Extra字段里的`Using index condition`表示使用了索引下推.通过字段`city`快速定位记录后,再直接利用索引中age字段的值来进行过滤.

这是MySQL5.6版本引入的**索引下推优化(index condition pushdown)**,可以在索引查找过程中,对索引包含的字段先做判断,直接过滤不满足条件的记录,减少回表次数,提高性能.

## 索引选择
主要是针对普通索引和唯一索引的选择.

在选择的前提是要能满足业务的需求,比如业务上需要保证唯一性,但应用层并不一定能保证,需要数据库层面来保证,就必须选择唯一索引.
又比如身份证号,在业务应用层面已经保证了唯一性,并不需要数据库层面来保证,就可以选择普通索引,当然也可以选择唯一索引.

在普通索引和唯一索引都能保证业务正确的前提下,如何评估该选择普通索引还是唯一索引列?下面主要从查询和更新两方面来进行评估.

### 查询
仍然以表`t`为例,针对字段`name`,分别创建唯一索引和普通索引时,其查找过程是如何的?针对查询语句:`select * from t where name='guanguan';`.
* 对于普通索引而言,查找到第一条满足条件的记录后,需要继续查找下一条记录,直到碰到第一个不满足条件的记录.
* 对于唯一索引而言,由于索引定义了唯一性,查找到第一个满足条件的记录后,就会停止继续检索.

上述两个查找过程的性能差异,几乎微乎其微.InnoDB读写都是基于数据页的,当要去磁盘查找某一记录时,是会把该记录所在的数据页整体读入内存的.对于普通索引的继续查找只是一次指针寻找和一次判断,对于现代CPU来说,影响微乎其微.

可见对于查询来说,普通索引和唯一索引之间的差异几乎微乎其微.

### 更新
在比较更新的差异之前，先引入概念`change buffer`,这也是InnoDB对更新操作所作出的优化.

当需要更新一个数据页时,如果该数据页在内存中,就直接更新内存的数据.但如果该数据页不在内存中,在不影响数据一致性的前提下,引擎会把这些更新操作缓存到`change buffer`中,这样就暂时不需要把磁盘数据读入到内存中,减少了磁盘随机读取,提升了更新性能.

在必要时,会把`change buffer`中的数据应用到原始数据页中,得到新的数据页,这个过程称为`merge`.

那`change buffer`中的数据什么时候应用到原始数据页中列?
* 当原始数据页被加载到Buffer pool时,会执行`merge`.
* 后台有线程定期会执行`merge`.
* 当MySQL正常关闭时,也会执行`merge`.

`change buffer`优化的前提是要不影响数据一致性,而对于唯一索引,由于需要判断是否唯一,就必须先把磁盘数据加载到内存中来进行判断,由于数据页已经在内存中了,直接更新内存就行了,所以该优化对唯一索引是不起作用的.因此`change buffer`优化只是对普通索引有效.

`change buffer`优化实际上是延迟了更新操作,当一个数据页上对应的更新操作在`change buffer`中越多,其收益是越大的.但如果针对先更新然后马上就查询的场景,这个优化可能会起到反作用.该场景下随机IO的次数不会少,反而增加了`change buffer`的维护代价.

`change buffer`用的是buffer pool里的内存,其大小可以通过参数`innodb_change_buffer_max_size`来动态设置,表示`change buffer`的大小最多只能占用buffer pool的比例.

另外是否启用`change buffer`或对哪些操作启用,是通过参数`innodb_change_buffering`来控制的,该参数默认是`all`,有以下几种选择:
- all: 默认值.开启buffer inserts、delete-marking operations、purges
- nono: 不开启
- inserts: 只对buffer insert操作(对insert和update有效)开启
- deletes: 只对delete-marking操作开启
- changes: 只对buffer insert和delete-marking操作开启
- purges: 只对在后台执行的物理删除操作开启.
``` sql
mysql> show variables like '%innodb_change_buffer%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| innodb_change_buffer_max_size | 25    |
| innodb_change_buffering       | all   |
+-------------------------------+-------+
2 rows in set (0.00 sec)
```

`change buffer`是会持久化的,保存在系统表空间(`ibdata1`)
* 数据库空闲时,后台有线程定时持久化
* 当数据库缓冲池空间不够时
* 当数据库正常关闭时
* redo log写满时

注意:
- `change buffer`也是通过B+树来存储的,键是表空间ID.
- `change buffer`的变更也是会记录redo日志的(**既然记录到redo中了,为什么还要持久化到系统表空间?**).
- `change buffer`节省的主要是随机读磁盘的IO消耗.

再来谈谈更新时,普通索引和唯一索引的差异:
* 当数据页在内存中时,唯一索引会校验下唯一性,通过后再更新内存;而普通索引直接更新内存即可.
* 当数据页不在内存中时,唯一索引会先通过磁盘随机读把数据加载到内存中,然后再校验唯一性,通过后再更新内存;而普通索引是直接把更新操作记录到`change buffer`中.

从更新来说,普通索引更具有优势.

## 为什么会选错索引?
### 现象分析
当一个表有多个索引,查询究竟走哪个索引?优化器是通过什么因素来决定使用哪个索引的?先看个例子.
``` sql
/*创建表t1*/
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB;

/*创建存储过程插入数据*/
delimiter ;;
create procedure idata() 
begin
  declare i int;
  declare j int;
  set i = 1;
  while(i <= 10) do 
    set j=1;
    start transaction;
    while(j<=10000) do 
      insert into t1(a, b) values(j+(i-1)*10000, j+(i-1)*10000); 
      set j=j+1; 
    end while;
    commit;
    set i=i+1;
  end while;
  end;;
delimiter ;

/*调用存储过程,插入数据到表中*/
call idata();

/*查询使用索引a,符合预期*/
mysql> explain select * from t1 where a between 10000 and 20000;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 5       | NULL | 10001 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
上面例子中,查询使用索引`a`,符合预期,说明优化器选择了正确的索引.

再来看看另一种场景,事务隔离级别为`REPEATABLE-READ`.
|session A|session B|
|--|--|
|start transaction with consistent snapshot;||
||delete from t1; call idata();|
||explain select * from t1 where a between 10000 and 20000;|
|commit;||

session A:
``` sql
mysql> explain select * from t1 where a between 10000 and 20000;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 5       | NULL | 10001 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

mysql> start transaction with consistent snapshot;
Query OK, 0 rows affected (0.00 sec)
```

session B:
``` sql
mysql> delete from t1;
Query OK, 100000 rows affected (0.72 sec)

mysql> call idata();
Query OK, 0 rows affected (2.93 sec)

mysql> explain select * from t1 where a between 10000 and 20000;
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL | 100015 |    37.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
session B此时选择的是全表扫描,开启慢查询日志,再来看看查询的具体信息.
``` sql
mysql> set long_query_time=0;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1 where a between 10000 and 20000;
mysql> select * from t1 force index(a) where a between 10000 and 20000;

/*慢查询日志,设置慢查询时间为0秒,即打印所有语句*/
set long_query_time=0;
# Time: 2020-06-16T06:28:51.528132Z
# User@Host: web[web] @ localhost []  Id:    57
# Query_time: 0.038091  Lock_time: 0.000289 Rows_sent: 10001  Rows_examined: 100000
SET timestamp=1592288931;
select * from t1 where a between 10000 and 20000;
# Time: 2020-06-16T06:29:01.998170Z
# User@Host: web[web] @ localhost []  Id:    57
# Query_time: 0.015590  Lock_time: 0.000345 Rows_sent: 10001  Rows_examined: 10001
SET timestamp=1592288941;
select * from t1 force index(a) where a between 10000 and 20000;
```
* 第一个查询语句的Rows_examined为100000行,走的是全表扫描,耗时38毫秒.
* 第二个查询语句的Rows_examined为10001行,走的是索引a,耗时15毫秒.
很显然,针对第一个查询语句,优化器选错了索引.

那么到底优化器选择索引的逻辑是什么列?

***优化器选择索引的目的,是找到一个最优执行方案,用最小的代价去执行语句.***

在数据库里面,扫描行数是影响执行代价的因素之一.扫描行数越少,意味这访问磁盘数据的次数越少,消耗CPU资源越少.但扫描行数并不是唯一的判断标准,优化器还会结合是否使用临时表、是否排序等因素进行综合判断.

**那扫描行数是怎么判断的?**
在执行语句之前,数据库并不能直到满足条件的记录到底有多少条?只能根据统计信息来估算.这个统计信息就是索引的区分度.一个索引上不同的值越多,这个索引的区分度就越好.一个索引上不同的值得个数,称之为基数(cardinality),基数越大,说明索引的区分度越好.
``` sql
/*查看索引基数*/
mysql> show index from t1;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t1    |          0 | PRIMARY  |            1 | id          | A         |      100015 |     NULL | NULL   |      | BTREE      |         |               |
| t1    |          1 | a        |            1 | a           | A         |      100015 |     NULL | NULL   | YES  | BTREE      |         |               |
| t1    |          1 | b        |            1 | b           | A         |      100015 |     NULL | NULL   | YES  | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
3 rows in set (0.00 sec)
```

索引的基数都是通过采样统计得来的,针对表做精确统计代价太大,只能是采样统计了.默认选择N个数据页,统计这些页面上的不同值,得到一个平均值,然后乘以这个索引的页面数,就得到了索引的基数.而数据表是会持续更新的,当变更的行数超过1/M时,会自动触发重新做一次索引统计.

InnoDB有两种存储索引统计的方式,通过参数`innodb_stats_persistent`的值来选择:
- 设置为on的时候,统计信息会持久化存储,默认的N是20,M是10
- 设置为off的时候,统计信息存储在内存中,默认的N是8,M是16

从`show index`结果可以看到,索引基数都是一样的,优化器还要判断这个语句本身要扫描多少行?
``` sql
mysql> explain select * from t1 where a between 10000 and 20000;
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL | 100015 |    37.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from t1 force index(a) where a between 10000 and 20000;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 5       | NULL | 37116 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
可以看到两个查询语句预计的扫描行数(rows字段),第一个语句的预估是正确的,rows的值是100015;但第二个语句的rows值是37116,偏差就大了(实际上rows应该是10001行).是这个偏差误导了优化器的判断.

第一个语句走的是主键索引,不需要回表;而第二个语句走的是普通索引,是需要回表查询的,回表的代价也是优化器需要考虑的.优化器认为直接扫描主键索引更快.

既然是统计信息不准确,那就需要修正,使用命令`analyze table t1`.可以看到修正之后,索引就选择正确了.
``` sql
mysql> analyze table t1;
+--------+---------+----------+----------+
| Table  | Op      | Msg_type | Msg_text |
+--------+---------+----------+----------+
| web.t1 | analyze | status   | OK       |
+--------+---------+----------+----------+
1 row in set (0.01 sec)

mysql> explain select * from t1 where a between 10000 and 20000;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 5       | NULL | 10001 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
1 row in set, 1 warning (0.01 sec)
```

再来看看另外一个语句:
``` sql
mysql> explain select * from t1 where (a between 1 and 1000) and (b between 50000 and 100000) order by b limit 1;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                              |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a,b           | b    | 5       | NULL | 50007 |     1.00 | Using index condition; Using where |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+------------------------------------+
1 row in set, 1 warning (0.01 sec)
```
优化器选择了索引b,rows为50007行,扫描行数的估计值不准确,有选错了索引.

### 选择异常的处理
1. 在语句中使用`force index`强制选择一个索引.
2. 考虑修改语句,引导MySQL使用我们期望的索引,把上面例子中的`order by b limit 1`换成`order by b, a limit 1`试试,看执行计划是怎样的.
3. 在某些场景下,可以新建一个更适合的索引来供优化器选择,或者删掉误用的索引.

### 思考
第一个例子中是通过session A和session B的配合,来复现选错了索引的情况,如果单独执行session B的语句,会出现什么情况?
``` sql
mysql> delete from t1;
Query OK, 100000 rows affected (0.55 sec)

mysql> call idata();
Query OK, 0 rows affected (2.41 sec)

mysql> explain select * from t1 where a between 10000 and 20000;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 5       | NULL | 10001 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
索引选择正确,为什么会出现这种差异?session A是马上开启一个一致性读视图,并没有其它操作,是因为什么造成了统计信息的错误?

当session A开启了一致性视图之后,session B的删除是不能直接把数据删除的.这样每一行数据会存在两个版本(`MVCC机制`),索引a上的数据其实是有两份的.但对于使用主键索引时,rows是直接按照表的行数来估计的,而表的行数,优化器是直接用`show table status`中的Rows值.
``` sql
mysql> show table status like 't1'\G
*************************** 1. row ***************************
           Name: t1
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 100256
 Avg_row_length: 36
    Data_length: 3686400
Max_data_length: 0
   Index_length: 3178496
      Data_free: 15728640
 Auto_increment: 300001
    Create_time: 2020-06-16 14:24:46
    Update_time: 2020-06-16 15:10:39
     Check_time: NULL
      Collation: utf8_unicode_ci
       Checksum: NULL
 Create_options: 
        Comment: 
1 row in set (0.00 sec)
```

## 如何给字符串字段加索引?
在字符串字段上加索引,是可以指定只取前几个字节的,如下:
``` sql
/*city全字段加索引*/
mysql> alter table t add index index1(name);
Query OK, 0 rows affected (0.48 sec)
Records: 0  Duplicates: 0  Warnings: 0

/*city字段前6个字节加索引*/
mysql> alter table t add index index2(name(5));
Query OK, 0 rows affected (0.48 sec)
Records: 0  Duplicates: 0  Warnings: 0
```
针对索引index2,占用的空间更小,这是前缀索引的优势,但带来的损失是可能会增加额外的记录扫描次数.
``` sql
/*先插入数据*/
mysql> insert into t (city, name, age) values ('hangzhou', 'zhangyi', 20);
mysql> insert into t (city, name, age) values ('hangzhou', 'zhanger', 20);
mysql> insert into t (city, name, age) values ('hangzhou', 'zhangsan', 20);
mysql> insert into t (city, name, age) values ('hangzhou', 'zhangsi', 20);
mysql> insert into t (city, name, age) values ('hangzhou', 'zhangwu', 20);

/*查询*/
mysql> select * from t where name = 'zhangsi';
+--------+----------+---------+-----+------+
| id     | city     | name    | age | addr |
+--------+----------+---------+-----+------+
| 128004 | hangzhou | zhangsi |  20 | NULL |
+--------+----------+---------+-----+------+
1 row in set (0.07 sec)
```
查询如果使用的是索引index1
1. 在索引树上能直接定位到索引值为`zhangsi`的记录,取得主键ID的值
2. 然后回表查询整行记录
3. 继续取下一个索引值,发现已不满足条件,循环结束.

查询如果使用的是索引index2
1. 在索引树上先定位到`zhang`的记录,取主键ID的值
2. 然后回表获取字段`name`的值发现不满足条件,丢弃
3. 继续取下一个索引值,值仍为`zhang`,取出主键ID的值
4. 重复步骤2,若满足条件就把行放入记录集中
5. 直到索引值不为`zhang`,循环结束.

使用索引index2,总共会有5次回表,扫描了5次.使用前缀索引,导致查询语句读数据的次数变多了.如果索引index2设置为`name(6)`列?此时只需要扫描2次了.

***使用前缀索引,定义好长度,就可以做到既节省空间,又不用额外增加太多的查询成本.***

建立索引时要关注索引的区分度,区分度越高越好.可以通过统计索引上有多少个不同的值来判断要使用多长的前缀.从如下sql可以看出,可以选用`name(4)`来作为索引.
``` sql
/*统计各个长度的前缀数量*/
mysql> select count(distinct name) as L, count(distinct left(name, 4)) as L4, count(distinct left(name, 5)) as L5, count(distinct left(name, 6)) as L6, count(distinct left(name, 7)) as L7 f
rom t;+-----+-----+-----+-----+-----+
| L   | L4  | L5  | L6  | L7  |
+-----+-----+-----+-----+-----+
| 149 | 145 | 145 | 148 | 149 |
+-----+-----+-----+-----+-----+
1 row in set (0.52 sec)
```

***当使用前缀索引时,会导致覆盖索引无效(无论前缀使用多少位).***

以`select id, name from t where name = 'zhangsi'`为例,如果使用索引index2(就算使用`name(16)`为索引),获取主键ID的值后还必须回表,然后判断name的值是否满足条件,这就导致覆盖索引无效了.

## 索引失效
### 条件字段函数操作
如果在`where`条件查询的字段上使用函数为如何?
``` sql
/*创建交易流水表*/
CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4

/*整个表模拟了10万行数据*/
mysql> select count(*) from tradelog;
+----------+
| count(*) |
+----------+
|   100000 |
+----------+
1 row in set (0.02 sec)

/*在字段t_modified使用month函数*/
mysql> explain select count(*) from tradelog where month(t_modified) = 7;
+----+-------------+----------+------------+-------+---------------+------------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | tradelog | NULL       | index | NULL          | t_modified | 6       | NULL | 100194 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
查询语句是统计所有7月份的流水,从Extra可以看出是使用了索引`t_modified`,从rows看是索引全扫描.使用函数`month`之后,获取到的值并不是有序的,所以无法利用索引的有序性来快速查找,只能是全索引扫描.

为了能利用索引的快速定位能力,就需要把上面的sql改造成按照字段本身的范围查询
``` sql
/*按照范围来查询,利用索引特性快速定位*/
mysql> explain select count(*) from tradelog where
    -> (t_modified >= '2016-07-01' and t_modified < '2016-08-01') or
    -> (t_modified >= '2017-07-01' and t_modified < '2017-08-01') or
    -> (t_modified >= '2018-07-01' and t_modified < '2018-08-01');
+----+-------------+----------+------------+-------+---------------+------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | tradelog | NULL       | range | t_modified    | t_modified | 6       | NULL | 5605 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

再来看看一个非常简单的加法操作,也是全表扫描,无法利用主键索引.
``` sql
mysql> explain select * from tradelog where id+1=1000;
+----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | tradelog | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 100194 |   100.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

***在索引字段上使用函数会导致无法利用索引的快速定位能力,不管是什么函数,都会导致优化器认为无法使用索引快速定位.***

### 隐式类型转换
``` sql
mysql> explain select * from tradelog where tradeid = 6981747220;
+----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | tradelog | NULL       | ALL  | tradeid       | NULL | NULL    | NULL | 100194 |    10.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 3 warnings (0.01 sec)
```
字段`tradeid`上是有索引的,而上面的语句直接走的主键全表扫描,为什么没有使用`tradeid`索引?

在表中字段`tradeid`的定义是`varchar(32)`,为字符串类型.而`where`里等号右边是个整数,当两边类型不一致时,MySQL是如何处理的?

当字符串与数字进行比较时,是把字符串转化为数字还是把数字转换为字符串?
``` sql
mysql> select "10" > 9;
+----------+
| "10" > 9 |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)
```
上面`select "10" > 9`返回1,说明是把字符串转换为数字了.

实际上语句`select * from tradelog where tradeid = 6981747220`相当于被转化为了`select * from tradelog where CAST(tradeid AS signed int) = 6981747220`,这条语句就触发了上面说的:对索引字段做函数操作,优化器放弃走树搜索功能.

### 隐式字符编码转换
``` sql
/*新建交易明细表*/
CREATE TABLE `trade_detail` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `trade_step` int(11) DEFAULT NULL,
  `step_info` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

/*关联查询*/
mysql> explain select * from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=100002;
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys   | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | l     | NULL       | const | PRIMARY,tradeid | PRIMARY | 4       | const |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | ALL   | NULL            | NULL    | NULL    | NULL  |   11 |   100.00 | Using where |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
驱动表为表`tradelog`,被驱动表为表`trade_detail`.查询对表`tradelog`是走的主键索引,但表`trade_detail`却是全表扫描,但在其字段`tradeid`是存在索引的,为何?

对比发现,表`tradelog`的字符集是`utf8mb4`,而表`trade_detail`的字符集是`utf8`,字符集不一样时查询时如何处理的?字符集`utf8mb4`是`utf8`的超集,MySQL会把`utf8`字符串转换为`utf8mb4`字符集,然后再做比较.查询语句会转化为如下:
``` sql
mysql> explain select * from tradelog l, trade_detail d where convert(d.tradeid USING utf8mb4)=l.tradeid and l.id=100002;
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys   | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | l     | NULL       | const | PRIMARY,tradeid | PRIMARY | 4       | const |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | ALL   | NULL            | NULL    | NULL    | NULL  |   11 |   100.00 | Using where |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
2 rows in set, 1 warning (0.01 sec)
```
转化后的语句对表`trade_detail`的索引上的字段做了函数操作,此时优化器是会放弃树搜索功能的,就导致做了全表扫描.

再来看看如下语句:
``` sql
mysql> explain select l.operator from tradelog l, trade_detail d where d.tradeid=l.tradeid and d.id=4;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | d     | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | l     | NULL       | ref   | tradeid       | tradeid | 131     | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```
驱动表为表`trade_detail`,被驱动表为表`tradelog`.这个语句里两个表都有用到索引,但驱动表用的是主键索引,而被驱动表用的是索引`tradeid`.语句在转化时是针对驱动表的`tradeid`字段,所以被驱动表可以用上索引`tradeid`.

针对字符集不一样的情况下的优化:
* 修改表结构,把字符集设置成一样.
* 修改sql语句,可以主动把驱动表的字符集修改为被驱动表的字符集,使得可以使用被驱动表的索引.
``` sql
mysql> explain select * from tradelog l, trade_detail d where d.tradeid=convert(l.tradeid using utf8) and l.id=100002;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | l     | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | d     | NULL       | ref   | tradeid       | tradeid | 99      | const |    4 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
2 rows in set, 1 warning (0.01 sec)
```

