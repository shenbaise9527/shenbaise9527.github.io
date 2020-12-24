# MySQL基于GTID复制


## 开启GTID.
``` 
# 启用gtid模式,每个事务有个唯一的id,全局事务ID,事务提交时分配,基于gtid来复制.
gtid_mode=ON

# 开启gtid的一些安全限制.
enforce_gtid_consistency=ON

# gtid生成方式,默认为自动.
# gtid_next=AUTOMATIC

# 从库启动复制,master_auto_position=1表示开启基于GTID的复制.
CHANGE MASTER TO MASTER_HOST='xxx', MASTER_PORT=3306, MASTER_USER='user_name', MASTER_PASSWORD='password', MASTER_AUTO_POSITION=1;
start slave;
```

## 并行复制参数.
``` 
# 并行复制类型,默认值为DATABASE,即按库来并行复制,LOGICAL_CLOCK为根据同时进入prepare和commit来并行复制.
slave_parallel_type=LOGICAL_CLOCK

# 并行复制线程数.
slave_parallel_workers=8

# 并行复制策略,默认值为COMMIT_ORDER,即按照上面的prepare和commit来并行;
# WRITESET对事务中的每一行计算hash,组合成writeset,如果两个事务没有更新相同行,writeset会没有交集可并行.
# WRITESET直接记录在binlog,不需要解析event,对binlog的格式没要求,5.7.22版本的新功能,binlog协议不向上兼容.
# WRITESET_SESSION,即在WRITESET基础上多了个约束,主库上同一线程先后执行的事务,在备库也要保证相同的顺序.
binlog_transaction_dependency_tracking=WRITESET
transaction_write_set_extraction=XXHASH64

# 记录writeset的容量,不需要修改,复制时可以并发的事务数大概为该值的一半.
#binlog_transaction_dependency_history_size=25000

# slave把从master接收到的binlog记录到自己的binlog中,主要用于级联复制的场景.
log_slave_updates=ON
```

## GTID的限制.
1. 从复制时报错,error: 1032
`select * from performance_schema.replication_applier_status_by_worker`可以查询从复制时的错误.
error: 1032,主删除数据,但从没有相应的记录.遇到错误主从复制会停止.

解决方案是在从库上跳过主库的这个事务:
``` 
-- 设置从库上的gtid_next为报错事务的gtid.
set gtid_next="af299bf7-dc7c-11ea-8417-0242ac170002:22805";
begin;
commit;
start slave;
```

2. create function报错,error: 1418
[Err] 1418 - This function has none of DETERMINISTIC, NO SQL, or READS SQL DATA in its declaration and binary logging is enabled (you might want to use the less safe log_bin_trust_function_creators variable)

解决方案:
``` 
DELIMITER ;;
CREATE FUNCTION `xxxx`(user_id INT) RETURNS varchar(4000) CHARSET utf8 COLLATE utf8_unicode_ci
-- 添加关键字DETERMINISTIC.
DETERMINISTIC
BEGIN
```

3. create table报错,error: 1786
[Err] 1786 - Statement violates GTID consistency: CREATE TABLE ... SELECT.

解决方案,需要拆分成两部分,create语句和insert语句:
``` 
CREATE TABLE xxxx LIKE t;
INSERT INTO xxxx SELECT * FROM t;
```

4. create temporary报错,error: 1787
[Err] 1787 - Statement violates GTID consistency: CREATE TEMPORARY TABLE and DROP TEMPORARY TABLE can only be executed outside transactional context.  These statements are also not allowed in a function or trigger because functions and triggers are also considered to be multi-statement transactions.

解决方案:
在`autocommit=1`的情况下可以创建临时表,主库创建临时表时不产生GTID信息,所以不会同步到从库,但在删除临时表时会产生GTID,从在处理时会报错,导致复制中断.

## 查看GTID
``` sql
-- 主库上执行,Executed_Gtid_Set表示已执行过的GTID集合.
mysql> show master status\G
*************************** 1. row ***************************
             File: bin.000005
         Position: 194
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: af299bf7-dc7c-11ea-8417-0242ac170002:1-22805
1 row in set (0.00 sec)

-- 在从库上执行,Executed_Gtid_Set表示从库已经执行过的GTID集合.
-- Retrieved_Gtid_Set,从库会扫描最后一个relay log,显示当前扫描所得的GTID集合.
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.20.151
                  Master_User: mtp2
                  Master_Port: 3406
                Connect_Retry: 60
              Master_Log_File: bin.000005
          Read_Master_Log_Pos: 194
               Relay_Log_File: relay_log.000007
                Relay_Log_Pos: 355
        Relay_Master_Log_File: bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 194
              Relay_Log_Space: 556
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: af299bf7-dc7c-11ea-8417-0242ac170002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 60012ca6-dc7d-11ea-8f34-0242ac180002:1-5,
af299bf7-dc7c-11ea-8417-0242ac170002:1-22805
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

-- 查看binlog event.在语句前后都会设置GTID_NEXT
-- SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count]
mysql> show binlog events in 'bin.000003' limit 0, 5;
+------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
| Log_name   | Pos | Event_type     | Server_id | End_log_pos | Info                                                              |
+------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
| bin.000003 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.31-log, Binlog ver: 4                             |
| bin.000003 | 123 | Previous_gtids |         1 |         194 | af299bf7-dc7c-11ea-8417-0242ac170002:1-5                          |
| bin.000003 | 194 | Gtid           |         1 |         259 | SET @@SESSION.GTID_NEXT= 'af299bf7-dc7c-11ea-8417-0242ac170002:6' |
| bin.000003 | 259 | Query          |         1 |         398 | create database mtp2 default charset utf8 collate utf8_unicode_ci |
| bin.000003 | 398 | Gtid           |         1 |         463 | SET @@SESSION.GTID_NEXT= 'af299bf7-dc7c-11ea-8417-0242ac170002:7' |
+------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
5 rows in set (0.00 sec)

-- 解析binlog.
root@baecf53ab2f8:/var/log/mysql# mysqlbinlog -vv bin.000003 --include-gtids='af299bf7-dc7c-11ea-8417-0242ac170002:6'
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#200812 17:17:57 server id 1  end_log_pos 123 CRC32 0x815f99b3 	Start: binlog v 4, server v 5.7.31-log created 200812 17:17:57 at startup
ROLLBACK/*!*/;
BINLOG '
xbMzXw8BAAAAdwAAAHsAAAAAAAQANS43LjMxLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAADFszNfEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AbOZX4E=
'/*!*/;
# at 123
#200812 17:17:57 server id 1  end_log_pos 194 CRC32 0x5856cea2 	Previous-GTIDs
# af299bf7-dc7c-11ea-8417-0242ac170002:1-5
# at 194
#200812 17:19:17 server id 1  end_log_pos 259 CRC32 0xf56e199c 	GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'af299bf7-dc7c-11ea-8417-0242ac170002:6'/*!*/;
# at 259
#200812 17:19:17 server id 1  end_log_pos 398 CRC32 0x010ed0fb 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1597223957/*!*/;
SET @@session.pseudo_thread_id=2/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C latin1 *//*!*/;
SET @@session.character_set_client=8,@@session.collation_connection=8,@@session.collation_server=192/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create database mtp2 default charset utf8 collate utf8_unicode_ci
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

