# MySQL的orderby分析


## 数据准备
``` sql
/*mysql版本*/
Server version: 5.7.27-log MySQL Community Server (GPL)

/*创建表t*/
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;

/*数据分布*/
mysql> select city, count(*) from t group by city;
+-----------+----------+
| city      | count(*) |
+-----------+----------+
| beijing   |     5590 |
| gongan    |     5398 |
| guangzhou |     5557 |
| hangzhou  |     5505 |
| ouchi     |     5402 |
| shanghai  |     5375 |
| shenzhen  |     5591 |
| wuhan     |     5582 |
+-----------+----------+
8 rows in set (0.02 sec)
```

## 带条件orderby过程分析
主要针对sql语句`select city, name, age from t where city='hangzhou' order by name limit 1000`来分析.

### 全字段排序
``` sql
mysql> explain select city, name, age from t where city='hangzhou' order by name limit 1000;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city          | city | 50      | const | 5505 |   100.00 | Using index condition; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+---------------------------------------+
1 row in set, 1 warning (0.00 sec)
```
通过执行计划可以看出，`rows`扫描行数为5505，符合预期。`Using index condition`(索引下推,Index Condition Pushdown)使用了索引`city`来遍历。`Using filesort`使用了排序.

开启optimizer_trace，来跟踪执行结果
``` sql
/*打开optimizer_trace，只对本线程有效*/
set optimizer_trace='enabled=on';

/*@a保存Innodb_rows_read的初始值*/
select variable_value into @a from performance_schema.session_status where variable_name='Innodb_rows_read';

/*执行sql语句*/
select city, name, age from t where city='hangzhou' order by name limit 1000;

/*查看optimizer_trace输出*/
select * from information_schema.OPTIMIZER_TRACE\G
/*截图部分结果*/
            "filesort_execution": [
            ],
            "filesort_summary": {
              "rows": 5505,
              "examined_rows": 5505,
              "number_of_tmp_files": 9,
              "sort_buffer_size": 262000,
              "sort_mode": "<sort_key, packed_additional_fields>"
            }

/*@b保存Innodb_rows_read的当前值*/
select variable_value into @b from performance_schema.session_status where variable_name='Innodb_rows_read';

/*计算Innodb_rows_read的差值*/
select @b-@a;

mysql> select @b-@a;
+-------+
| @b-@a |
+-------+
|  5506 |
+-------+
1 row in set (0.00 sec)
```
optimizer_trace结果分析：
* optimizer_trace结果中的`number_of_tmp_files`可以看到使用了临时文件来排序.说明排序过程中内存放不下所有待排序数据,需要使用外部排序(一般采用归并排序).
* Mysql把待排序的数据分成了9份,每一份单独排序存放在临时文件中,最后把这9个有序文件再合并成一个有序的大文件.
* 若`number_of_tmp_files`为0,表示排序是在内存中完成的.Mysql通过参数`sort_buffer_size`来定义排序缓存的大小,若`sort_buffer_size`越小,需要分成的份数就越多.
* optimizer_trace结果中的`examined_rows`表示参与排序的行数.
* optimizer_trace结果中的`sort_mode`里的`packed_additional_fields`表示对字符串进行了紧凑处理,针对varchar字段是按照实际长度来分配空间的.
* `packed_additional_fields`也表明排序采用的是全字段排序,即排序时包含所有查询字段(先把city、name和age字段查询出来放入到临时文件中,然后再根据name排序).Mysql通过参数`max_length_for_sort_data`来控制排序字段的长度,默认是1024.

整个排序过程:
1. 根据参数`max_length_for_sort_data`来判断,放入sort_buffer的字段(字段为city、name和age).
2. 从索引`city`中找到满足条件的主键id.
3. 根据主键id到主键索引中取出整行数据,取city、name、age三个字段的值,存入sort_buffer中.
4. 从索引`city`中获取下一个满足条件的主键id.
5. 循环3、4直到city不满足条件为止.
6. 对sort_buffer中的数据按照字段name做排序.
7. 排序完成sort_buffer内存空间就会被释放.

`Innodb_rows_read`的差值为什么是5506而不是5505？

因为在查询`OPTIMIZER_TRACE`表时，需要用到临时表，而临时表的存储引擎默认是InnoDB(通过参数`internal_tmp_disk_storage_engine`来控制的)，再把数据从临时表取出来时，会让Innodb_rows_read的值加1

### rowid排序
若查询的字段很多,总长度超过了`max_length_for_sort_data`所规定的长度,排序的过程是如何的?

修改参数`set max_length_for_sort_data=16;`,查询的三个字段的总长度为为36,则可触发上面说的情况.然后再来看看排序的过程.
``` sql
/*@a保存Innodb_rows_read的初始值*/
select variable_value into @a from performance_schema.session_status where variable_name='Innodb_rows_read';

/*执行sql语句*/
select city, name, age from t where city='hangzhou' order by name limit 1000;

/*查看optimizer_trace输出*/
select * from information_schema.OPTIMIZER_TRACE\G
/*截图部分结果*/
            "filesort_execution": [
            ],
            "filesort_summary": {
              "rows": 5505,
              "examined_rows": 5505,
              "number_of_tmp_files": 9,
              "sort_buffer_size": 261760,
              "sort_mode": "<sort_key, rowid>"
            }

/*@b保存Innodb_rows_read的当前值*/
select variable_value into @b from performance_schema.session_status where variable_name='Innodb_rows_read';

/*计算Innodb_rows_read的差值*/
select @b-@a;

mysql> select @b-@a;
+-------+
| @b-@a |
+-------+
|  6506 |
+-------+
1 row in set (0.00 sec)
```
optimizer_trace结果分析:
* 依然会采用外部排序,使用了9个临时文件来排序
* `sort_mode`变更为`rowid`,表明排序时的列只有要排序的列(name字段)和主键id.根据name排序完后还要根据对应的主键id去获取字段的值.
* rowid排序比全字段排序会多了回表操作,必定会影响排序的性能.
* `select @b-@a`的结果为6506,比之前的多了1000,为什么?
  - 全字段排序时回表是在引擎层内部自动完成的,server层并不感知,server层只是调用了5505次引擎的读接口获取city、name、age的值,然后在server层完成排序,所以是5505次读.
  - rowid排序时server层会先调用5505次引擎的读接口获取name、id的值,然后在server层完成排序,然后取前1000条记录中的id调用引擎的读接口获取对应的city、name、age的值,所以是6505次读.

整个排序过程:
1. 根据参数`max_length_for_sort_data`来判断,放入sort_buffer的字段(字段为name和主键id).
2. 从索引`city`中找到满足条件的主键id.
3. 根据主键id到主键索引中取出整行数据,取id和name字段的值,存入sort_buffer中.
4. 从索引`city`中获取下一个满足条件的主键id.
5. 循环3、4直到city不满足条件为止.
6. 对sort_buffer中的数据按照字段name做排序.
7. 再根据排序结果中的主键id去主键索引中获取city、name和age字段的值.

### 利用索引有序的特性
`order by name`需要排序是因为`name`字段是无序的,如果为有序的,`order by`又是怎样处理的?
``` sql
/*增加city+name的索引*/
mysql> alter table t add index city_name(city, name);
Query OK, 0 rows affected (0.34 sec)
Records: 0  Duplicates: 0  Warnings: 0

/*查看执行计划*/
mysql> explain select city, name, age from t where city='hangzhou' order by name limit 1000;
+----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys  | key       | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city,city_name | city_name | 50      | const | 5505 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
可以看到执行计划Extra中只有`Using index condition`,使用索引`city_name`而不用再排序了.虽然不用再排序,但仍然需要回表操作来获取city、name和age的值.

如果使用了覆盖索引,该查询语句又会是怎样的列?
``` sql
/*增加city+name+age的索引,覆盖所有查询字段*/
mysql> alter table t add index city_name_age(city, name, age);
Query OK, 0 rows affected (0.32 sec)
Records: 0  Duplicates: 0  Warnings: 0

/*查看执行计划*/
mysql> explain select city, name, age from t where city='hangzhou' order by name limit 1000;
+----+-------------+-------+------------+------+------------------------------+-----------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys                | key       | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+------------------------------+-----------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city,city_name,city_name_age | city_name | 50      | const | 5505 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+------------------------------+-----------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
从执行计划中可以看出仍然是使用的索引`city_name`,为什么不使用索引`city_name_age`来避免回表操作列?

强制使用索引`city_name_age`来看看具体情况.
``` sql
mysql> explain select city, name, age from t force index (city_name_age) where city='hangzhou' order by name limit 1000;
+----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys | key           | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city_name_age | city_name_age | 50      | const | 9940 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.01 sec)
```
从执行计划中可以看出使用覆盖索引(Extra中的`Using index`就是覆盖索引).**但rows为什么是9940?**

再来看看使用`order by name, age`的情况.
``` sql
mysql> explain select city, name, age from t where city='hangzhou' order by name, age limit 1000;
+----+-------------+-------+------------+------+------------------------------+---------------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys                | key           | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+------------------------------+---------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city,city_name,city_name_age | city_name_age | 50      | const | 5505 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+------------------------------+---------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
这时和预期是一样的,使用了索引city_name_age(覆盖索引`Using index`).

再来看看当把索引`city_name`删除后的情况.
``` sql
/*删除索引city_name*/
mysql> alter table t drop index city_name;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select city, name, age from t where city='hangzhou' order by name limit 1000;
+----+-------------+-------+------------+------+--------------------+---------------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys      | key           | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+--------------------+---------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city,city_name_age | city_name_age | 50      | const | 5505 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+--------------------+---------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
可以看到查询使用了索引`city_name_age`(覆盖索引`Using index`),避免了回表操作.

**注意,针对`order by name`,在使用索引`city_name`和`city_name_age`时在语义上是有细微差异的,两个查询返回的结果可能并不一样,这是两个索引的排序规则不一样导致的.**
``` sql
mysql> select city, name, age from t force index (city_name_age) where city='hangzhou' order by name limit 5;
+----------+------+-----+
| city     | name | age |
+----------+------+-----+
| hangzhou | ??   |   8 |
| hangzhou | ??   |  10 |
| hangzhou | ??   |  10 |
| hangzhou | ??   |  10 |
| hangzhou | ??   |  13 |
+----------+------+-----+
5 rows in set (0.00 sec)

mysql> select city, name, age from t  where city='hangzhou' order by name limit 5;
+----------+------+-----+
| city     | name | age |
+----------+------+-----+
| hangzhou | ??   |  82 |
| hangzhou | ??   |  57 |
| hangzhou | ??   |  29 |
| hangzhou | ??   |  86 |
| hangzhou | ??   |  16 |
+----------+------+-----+
5 rows in set (0.00 sec)
```
可以看出查询结果明显不一样.

**但如何解释同时存在索引`city_name`和`city_name_age`时,查询为什么不优先使用索引`city_name_age`来避免回表操作,而强制使用`city_name_age`时explain中rows为什么会是9940?**

## 不带条件的orderby分析
针对语句`select * from t order by city limit 1000`,会使用索引吗?如果会是哪个索引?

针对语句`select * from t order by city limit 10000`列?
``` sql
mysql> explain select * from t order by city limit 1000;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | index | NULL          | city | 50      | NULL | 1000 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from t order by city limit 10000;
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+----------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 44064 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```
从执行计划中可以看出,语句1是有使用索引`city`且不用排序.而语句2是全表扫描且使用了排序(Extra的`Using filesort`).

当把limit从1000改成10000时查询为什么造成这种差异?
* 使用索引`city`是需要进行回表操作的,当limit数据量大时优化器认为回表操作代价太大,还不如直接全表扫描.
* 针对回表操作,从索引`city`中获取到的id不是有序的,回表会造成随机读,这也是优化器认为代价太大的原因(Mysql其实有提供MRR机制来优化这种情况),还不如直接全表扫描,使用顺序读.

## 总结
* 可以通过合理的创建索引来避免`order by`排序,提高查询性能.
* 当排序不可避免时,有两个系统参数`max_length_for_sort_data`和`sort_buffer_size`会对排序过程产生影响.
* 当排序不可避免时,尽量使用`sort_buffer`内存+全字段排序,这样性能最好.可以考虑优化上面两个参数.
* Mysql设计思想之一:当内存足够时,就要多利用内存,尽量减少磁盘访问.
* Mysql设计思想之一:尽量避免磁盘随机IO.
* `Using filesort`表示需要排序.
* `Using where; Using index`表示使用覆盖索引,需要的数据都在索引列中,不需要回表.
* `Using index condition`表示索引下推来过滤条件,但需要回表查询数据.
* `Using where`表示使用索引的情况下,需要回表查询数据.

