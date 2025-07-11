# 12 MySQL 问题追踪与性能优化


MySQL 中怎么做问题追踪与性能优化

<!-- more -->


## 1. 问题排查的工具
mysql 中有以下几种问题排查的工具:
1. `show processlist`
2. `show engine innodb status`
3. `information_schema.innodb_trx`
4. `show engine innodb status`
4. `optimizer_trace`
5. 慢查询日志
6. `performance_schema 和 sys 系统库`

### 1.1 试验环境
接下来我们以下面的实现环境看看，如何使用这些工具排查 MySQL 的性能问题

```bash

mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000) do
    insert into t values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

### 1.2 Mariadb 的差异
Mariadb 没有 sys schema 库，mysql sys schema 部分功能通过 information_schema 和 插件提供。下面是二者之间的简单对比:

|功能|MySQL|Mariadb|
|:---|:---|:---|
|查看MDL锁|sys.schema_table_lock_waits|INSTALL SONAME 'metadata_lock_info'<br>information_schema.metadata_lock_info|
|查看Innodb行锁|sys.innodb_lock_waits|information_schema.innodb_lock_waits<br>INNODB_LOCKS<br>INNODB_TRX|
|会话状态|performance_schema.session_status|information_schema.session_status|


## 2. 锁等待排查
在表 t 执行 SQL 语句 `select * from t where id=1;` 如果查询长时间不返回，一般这种情况，大概率是表 t 被锁住了。接下来分析原因的时候，一般都是首先执行一下 show processlist 命令，看看当前语句处于什么状态。然后我们再针对每种状态，去分析它们产生的原因、如何复现，以及如何处理。

出现锁等待有以下几种情况:
1. Waiting for table metadata lock，即 等待 MDL 锁
2. Waiting for table flush
3. Innodb 行锁

### 2.1 等待 MDL 锁
第一种情况如下图所示，查询正在等待 MDL 锁。

![show_processlist](/images/mysql/MySQL45讲/show_processlist.png)

这类问题的处理方式，就是找到谁持有 MDL 写锁，然后把它 kill 掉。但是，由于在 show processlist 的结果里面，持有 MDL 写锁线程的 Command 列可能是“Sleep”，导致查找起来很不方便。

通过查询 sys.schema_table_lock_waits 这张表，我们就可以直接找出造成阻塞的 process id，把这个连接用 kill 命令断开即可。

![schema_table_lock_waits](/images/mysql/MySQL45讲/schema_table_lock_waits.png)


不过启用 performance_schema 和 sys(mariadb 中为 information_schema) 系统库需要 MySQL 在启动时设置 `performance_schema=on`，相比于设置为 off 会有 10% 左右的性能损失。

#### mariadb 
mariadb 没有 sys.schema_table_lock_waits 可以通过 METADATA_LOCK_INFO plugin 来查看MDL 锁的相关信息。
```bash
# 1. 启用 METADATA_LOCK_INFO plugin
# 临时
INSTALL SONAME 'metadata_lock_info';

# 永久
[mariadb]
...
plugin_load_add = metadata_lock_info

# 2. 查看MDL锁
MariaDB [performance_schema]> SELECT * FROM information_schema.metadata_lock_info;
+-----------+--------------------------+---------------+----------------------+--------------+------------+
| THREAD_ID | LOCK_MODE                | LOCK_DURATION | LOCK_TYPE            | TABLE_SCHEMA | TABLE_NAME |
+-----------+--------------------------+---------------+----------------------+--------------+------------+
|        11 | MDL_BACKUP_DDL           | NULL          | Backup lock          |              |            |
|        11 | MDL_SHARED_NO_READ_WRITE | NULL          | Table metadata lock  | tsong        | words      |
|        11 | MDL_INTENTION_EXCLUSIVE  | NULL          | Schema metadata lock | tsong        |            |
+-----------+--------------------------+---------------+----------------------+--------------+------------+
3 rows in set (0.000 sec)

# 直接查看被锁住的线程
MariaDB [performance_schema]> SELECT
    -> CONCAT('Thread ',P.ID,' executing "',P.INFO,'" IS LOCKED BY Thread ',
    -> M.THREAD_ID) WhoLocksWho
    -> FROM INFORMATION_SCHEMA.PROCESSLIST P,
    -> INFORMATION_SCHEMA.METADATA_LOCK_INFO M
    -> WHERE LOCATE(lcase(LOCK_TYPE), lcase(STATE))>0;
+----------------------------------------------------------------------------------------+
| WhoLocksWho                                                                            |
+----------------------------------------------------------------------------------------+
| Thread 10 executing "select * from words where id=1 for update" IS LOCKED BY Thread 11 |
+----------------------------------------------------------------------------------------+
1 row in set (0.002 sec)

# kill 占用锁的线程
MariaDB [performance_schema]> kill 11
```


### 2.2 等 flush
第二种情况如下图所示，查询在等 flush

![wait_flush](/images/mysql/MySQL45讲/wait_flush.png)

flush tables 有两种用法:
- flush tables t with read lock: 只关闭表 t
- flush tables with read lock: 关闭 MySQL 里所有打开的表。
- 正常这两个语句执行起来都很快，除非它们也被别的线程堵住了

出现上面 Waiting for table flush 状态情况是：有一个 flush tables 命令被别的语句堵住了，然后它又堵住了我们的 select 语句。


### 2.3 innodb 行锁
第三种情况，等待Innodb 行锁的情况如下:

![line_lock](/images/mysql/MySQL45讲/line_lock.png)

state 为 statistics 的线程被阻塞。这个问题并不难分析，但问题是怎么查出是谁占着这个写锁。如果你用的是 MySQL 5.7 版本，可以通过 sys.innodb_lock_waits 表查到。

![innodb_lock_waits](/images/mysql/MySQL45讲/innodb_lock_waits.png)

可以看到，4 号线程是造成堵塞的罪魁祸首。而干掉这个罪魁祸首的方式，就是 KILL QUERY 4 或 KILL 4。
- 不过，这里不应该显示“KILL QUERY 4”。这个命令表示停止 4 号线程当前正在执行的语句，而这个方法其实是没有用的。因为占有行锁的是 update 语句，这个语句已经是之前执行完成了的，现在执行 KILL QUERY，无法让这个事务去掉 id=1 上的行锁。
- 实际上，KILL 4 才有效，也就是说直接断开这个连接。这里隐含的一个逻辑就是，连接被断开的时候，会自动回滚这个连接里面正在执行的线程，也就释放了 id=1 上的行锁。

#### mariadb 
mariadb innodb_lock_waits 表位于 information_schema 中。可以使用下面的方法查看等待 Innodb 行锁的线程。 

```bash
# 1. 查看事务持有的锁
MariaDB [information_schema]> SELECT l.*, t.*
    FROM information_schema.INNODB_LOCKS l
    JOIN information_schema.INNODB_TRX t
        ON l.lock_trx_id = t.trx_id
    WHERE trx_state = 'LOCK WAIT' \G

# 2. 查看行锁等待，
MariaDB [information_schema]>  SELECT requesting_trx_id     AS run_trx_id, 
       r.trx_mysql_thread_id AS run_pid, 
       blocking_trx_id, 
       z.trx_mysql_thread_id AS blocking_pid 
FROM   innodb_lock_waits AS l 
       JOIN innodb_trx AS r 
         ON l.requesting_trx_id = r.trx_id 
       JOIN innodb_trx AS z 
         ON l.blocking_trx_id = z.trx_id;

+-----------------+---------+-----------------+--------------+
| run_trx_id      | run_pid | blocking_trx_id | blocking_pid |
+-----------------+---------+-----------------+--------------+
| 422151806689784 |      13 | 21240           |           10 |
+-----------------+---------+-----------------+--------------+
1 row in set (0.001 sec)

```

### 2.4 死锁查看
出现死锁后，执行 show engine innodb status，有一节 LATESTDETECTED DEADLOCK，就是记录的最后一次死锁信息。

并发执行下面两个语句，产生的死锁信息如下图所示:
```bash
select id from t where c in(5,20,10) lock in share mode;
select id from t where c in(5,20,10) order by c desc for update;
```

![innodb_status](/images/mysql/MySQL45讲/innodb_status.png)

1. 这个结果分成三部分：
  - (1) TRANSACTION，是第一个事务的信息；
  - (2) TRANSACTION，是第二个事务的信息；
  - WE ROLL BACK TRANSACTION (1)，是最终的处理结果，表示回滚了第一个事务。
2. 第一个事务的信息中：
  - WAITING FOR THIS LOCK TO BE GRANTED，表示的是这个事务在等待的锁信息；
  - index c of table `test`.`t`，说明在等的是表 t 的索引 c 上面的锁；
  - lock mode S waiting 表示这个语句要自己加一个读锁，当前的状态是等待中；
  - Record lock 说明这是一个记录锁；
  - n_fields 2 表示这个记录是两列，也就是字段 c 和主键字段 id；
  - 0: len 4; hex 0000000a; asc ;; 是第一个字段，也就是 c。值是十六进制 a，也就是 10；
  - 1: len 4; hex 0000000a; asc ;; 是第二个字段，也就是主键 id，值也是 10；
  - 这两行里面的 asc 表示的是，接下来要打印出值里面的“可打印字符”，但 10 不是可打印字符，因此就显示空格。
  - 第一个事务信息就只显示出了等锁的状态，在等待 (c=10,id=10) 这一行的锁。当然你是知道的，既然出现死锁了，就表示这个事务也占有别的锁，但是没有显示出来。别着急，我们从第二个事务的信息中推导出来。
3. 第二个事务显示的信息要多一些：
  - “ HOLDS THE LOCK(S)”用来显示这个事务持有哪些锁；
  - index c of table `test`.`t` 表示锁是在表 t 的索引 c 上；
  - hex 0000000a 和 hex 00000014 表示这个事务持有 c=10 和 c=20 这两个记录锁；
  - WAITING FOR THIS LOCK TO BE GRANTED，表示在等 (c=5,id=5) 这个记录锁。

4. 说明:
  - lock_mode X/S waiting表示next-key lock；
  - lock_mode X/S locks rec but not gap是只有行锁；
  - locks gap before rec，就是只有间隙锁；

从上面这些信息中，我们就知道：
  - “lock in share mode”的这条语句，持有 c=5 的记录锁，在等 c=10 的锁；
  - “for update”这个语句，持有 c=20 和 c=10 的记录锁，在等 c=5 的记录锁。

因此导致了死锁。这里，我们可以得到两个结论：
- 由于锁是一个个加的，要避免死锁，对同一组资源，要按照尽量相同的顺序访问；
- 在发生死锁的时刻，for update 这条语句占有的资源更多，回滚成本更大，所以 InnoDB 选择了回滚成本更小的 lock in share mode 语句，来回滚。


## 2. SQL执行追踪
可以用下面介绍的方法，来确定一个排序语句是否使用了临时文件，以及使用的排序算法。

```bash
/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* mariadb */
select VARIABLE_VALUE into @a from  information_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

## 3. 性能优化
说完了 MySQL 一些常见的问题追踪，接下来，我们来看看一些在业务高峰期拿来"应急"的解决方案，并着重说一说它们可能存在的风险，内容包括:
1. 短连接风暴
2. 慢查询性能问题
3. QPS 突增问题

## 4. 短连接风暴
正常的短连接模式就是连接到数据库后，执行很少的 SQL 语句就断开，下次需要的时候再重连。如果使用的是短连接，在业务高峰期的时候，就可能出现连接数突然暴涨的情况。

前面我们说过，MySQL 建立连接的过程，成本是很高的。除了正常的网络连接三次握手外，还需要做登录权限判断和获得这个连接的数据读写权限。在数据库压力比较小的时候，这些额外的成本并不明显。但是，短连接模型存在一个风险，就是一旦数据库处理得慢一些，连接数就会暴涨。

max_connections 参数，用来控制一个 MySQL 实例同时存在的连接数的上限，超过这个值，系统就会拒绝接下来的连接请求，并报错提示“Too many connections”。在机器负载比较高的时候，处理现有请求的时间变长，每个连接保持的时间也更长。这时，再有新建连接的话，就可能会超过 max_connections 的限制。对于被拒绝连接的请求来说，从业务角度看就是数据库不可用。

显然我们不能直接增大 max_connections 的值，让更多的连接都可以进来。因为更多的连接会进一步增大系统负载。，大量的资源耗费在权限验证等逻辑上，结果可能是适得其反，已经连接的线程拿不到 CPU 资源去执行业务的 SQL 请求。

这里还有两种方法，但要注意，这些方法都是有损的:
1. 第一种方法：先处理掉那些占着连接但是不工作的线程。
2. 减少连接过程的消耗

### 4.1 先处理掉那些占着连接但是不工作的线程
对于那些不需要保持的连接，我们可以通过 kill connection 主动踢掉。这个行为跟事先设置 wait_timeout 的效果是一样的。设置 wait_timeout 参数表示的是，一个线程空闲 wait_timeout 这么多秒之后，就会被 MySQL 直接断开连接。

但是需要注意，在 show processlist 的结果里，踢掉显示为 sleep 的线程，可能是有损的。我们来看下面这个例子。


![show_processlist_sleep](/images/mysql/MySQL45讲/show_processlist_sleep.png)

session A,B 在show processlist 中都是 Sleep，但按照优先级来说，应该优先断开像 session B 这样的事务外空闲的连接。但是，怎么判断哪些是事务外空闲的呢？可以通过 information_schema 库的 innodb_trx 表。因此，如果是连接数过多，你可以优先断开事务外空闲太久的连接；如果这样还不够，再考虑断开事务内空闲太久的连接。


从服务端断开连接使用的是 kill connection + id 的命令， 一个客户端处于 sleep 状态时，它的连接被服务端主动断开后，这个客户端并不会马上知道。直到客户端在发起下一个请求的时候，才会收到这样的报错“ERROR 2013 (HY000): Lost connection to MySQL server during query”。

从数据库端主动断开连接可能是有损的，尤其是有的应用端收到这个错误后，不重新连接，而是直接用这个已经不能用的句柄重试查询。这会导致从应用端看上去，“MySQL 一直没恢复”。

### 4.2 减少连接过程的消耗
有的业务代码会在短时间内先大量申请数据库连接做备用，如果现在数据库确认是被连接行为打挂了，那么一种可能的做法，是让数据库跳过权限验证阶段。但是风险极高，非常不建议。

跳过权限验证的方法是：重启数据库，并使用–skip-grant-tables 参数启动。在 MySQL 8.0 版本里，如果你启用–skip-grant-tables 参数，MySQL 会默认把 --skip-networking 参数打开，表示这时候数据库只能被本地的客户端连接。

## 5. 慢查询性能问题
在 MySQL 中，会引发性能问题的慢查询，大体有以下三种可能：
1. 索引没有设计好；
2. SQL 语句没写好；
3. MySQL 选错了索引。

### 5.1 索引没设计好
这种场景一般就是通过紧急创建索引来解决。MySQL 5.6 版本以后，创建索引都支持 Online DDL 了，对于那种高峰期数据库已经被这个语句打挂了的情况，最高效的做法就是直接执行 alter table 语句。

比较理想的是能够在备库先执行。假设你现在的服务是一主一备，主库 A、备库 B，这个方案的大致流程是这样的：
1. 在备库 B 上执行 set sql_log_bin=off，也就是不写 binlog，然后执行 alter table 语句加上索引；
2. 执行主备切换；
3. 这时候主库是 B，备库是 A。在 A 上执行 set sql_log_bin=off，然后执行 alter table 语句加上索引。

这是一个“古老”的 DDL 方案。平时在做变更的时候，你应该考虑类似 gh-ost 这样的方案，更加稳妥。但是在需要紧急处理时，上面这个方案的效率是最高的。

### 5.2 语句没写好
我们可以通过改写 SQL 语句来处理。MySQL 5.7 提供了 query_rewrite 功能，可以把输入的一种语句改写成另外一种模式。

```bash
mysql> insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");

call query_rewrite.flush_rewrite_rules();
```
call query_rewrite.flush_rewrite_rules() 这个存储过程，是让插入的新规则生效。

### 5.3 选错了索引
应急方案就是给这个语句加上 force index。同样地，使用查询重写功能，给原来的语句加上 force index，也可以解决这个问题。

### 5.4 问题总结
慢查询导致性能问题的三种可能情况，实际上出现最多的是前两种，即：索引没设计好和语句没写好。而这两种情况，恰恰是完全可以避免的。比如，通过下面这个过程，我们就可以预先发现问题。
1. 上线前，在测试环境，把慢查询日志（slow log）打开，并且把 long_query_time 设置成 0，确保每个语句都会被记录入慢查询日志；
2. 在测试表里插入模拟线上的数据，做一遍回归测试；
3. 观察慢查询日志里每类语句的输出，特别留意 Rows_examined 字段是否与预期一致。

如果新增的 SQL 语句不多，手动跑一下就可以。而如果是新项目的话，或者是修改了原有项目的 表结构设计，全量回归测试都是必要的。这时候，你需要工具帮你检查所有的 SQL 语句的返回结果。比如，你可以使用开源工具 [pt-query-digest](https://www.percona.com/doc/percona-toolkit/3.0/pt-query-digest.html)。

## 6. QPS 突增问题
有时候由于业务突然出现高峰，或者应用程序 bug，导致某个语句的 QPS 突然暴涨，也可能导致 MySQL 压力过大，影响服务。如果是bug 引起的，最理想的情况是让业务把这个功能下掉，服务自然就会恢复。

而下掉一个功能，如果从数据库端处理的话，对应于不同的背景，有不同的方法可用。我这里再和你展开说明一下:
1. 一种是由全新业务的 bug 导致的。假设你的 DB 运维是比较规范的，也就是说白名单是一个个加的。这种情况下，如果你能够确定业务方会下掉这个功能，只是时间上没那么快，那么就可以从数据库端直接把白名单去掉。
2. 如果这个新功能使用的是单独的数据库用户，可以用管理员账号把这个用户删掉，然后断开现有连接。这样，这个新功能的连接不成功，由它引发的 QPS 就会变成 0。
3. 如果这个新增的功能跟主体功能是部署在一起的，那么我们只能通过处理语句来限制。这时，我们可以使用上面提到的查询重写功能，把压力最大的 SQL 语句直接重写成"select 1"返回。

当然，这个操作的风险很高，需要你特别细致。它可能存在两个副作用：
1. 如果别的功能里面也用到了这个 SQL 语句模板，会有误伤；
2. 很多业务并不是靠这一个语句就能完成逻辑的，所以如果单独把这一个语句以 select 1 的结果返回的话，可能会导致后面的业务逻辑一起失败。

所以，方案 3 是用于止血的，跟前面提到的去掉权限验证一样，应该是你所有选项里优先级最低的一个方案。

同时你会发现，其实方案 1 和 2 都要依赖于规范的运维体系：虚拟化、白名单机制、业务账号分离。由此可见，更多的准备，往往意味着更稳定的系统。

在实际开发中，我们也要尽量避免一些低效的方法，比如避免大量地使用短连接。同时，如果你做业务开发的话，要知道，连接异常断开是常有的事，你的代码里要有正确地重连并重试的机制。

## 7. MySQL 监控

### 7.1 死锁监控
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 最近一次死锁详情          | `information_schema.INNODB_TRX`         | `SHOW ENGINE INNODB STATUS;` (查看 **LATEST DETECTED DEADLOCK** 部分)    |
| 当前阻塞中的事务          | `information_schema.INNODB_TRX`         | ```SELECT * FROM information_schema.INNODB_TRX WHERE TRX_STATE = 'LOCK WAIT'\G``` |
| 锁等待关系链 (8.0+)       | `performance_schema.data_lock_waits`    | ```SELECT * FROM performance_schema.data_lock_waits\G```                 |
| 当前持有的锁详情 (8.0+)   | `performance_schema.data_locks`         | ```SELECT * FROM performance_schema.data_locks\G```                      |

---

### 7.2 事务监控
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 运行时间 > 60s 的长事务   | `information_schema.INNODB_TRX`         | ```SELECT * FROM information_schema.INNODB_TRX WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60\G``` |
| 事务隔离级别状态          | `performance_schema.variables_by_thread`| ```SELECT * FROM variables_by_thread WHERE VARIABLE_NAME IN ('transaction_isolation','tx_isolation')\G``` |
| 未提交事务的修改行数      | `information_schema.INNODB_TRX`         | ```SELECT trx_id, trx_rows_modified FROM information_schema.INNODB_TRX WHERE trx_state = 'RUNNING'\G``` |
| 事务锁等待超时时间        | `performance_schema.global_variables`   | ```SELECT * FROM global_variables WHERE VARIABLE_NAME = 'innodb_lock_wait_timeout'\G``` |

---

### 7.3 性能监控
#### 查询级性能
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 慢查询日志状态            | `performance_schema.global_variables`   | ```SELECT * FROM global_variables WHERE VARIABLE_NAME LIKE 'slow_query_log%'\G``` |
| 慢查询统计 (8.0+)         | `sys.x$statement_analysis`              | ```SELECT * FROM sys.x$statement_analysis ORDER BY avg_latency DESC LIMIT 10\G``` |
| 全表扫描的语句            | `sys.schema_tables_with_full_table_scans`| ```SELECT * FROM sys.schema_tables_with_full_table_scans\G```           |
| 等待锁的语句统计          | `performance_schema.events_waits_summary_by_thread_by_event_name` | ```SELECT * FROM events_waits_summary_by_thread_by_event_name WHERE SUM_TIMER_WAIT > 0\G``` |

#### 连接与会话
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 当前活跃会话详情          | `performance_schema.processlist`        | ```SELECT * FROM processlist WHERE COMMAND != 'Sleep'\G```              |
| 连接数统计                | `performance_schema.threads`            | ```SELECT USER, COUNT(*) FROM threads GROUP BY USER\G```               |
| 会话内存使用 (8.0+)       | `sys.session`                           | ```SELECT * FROM sys.session ORDER BY current_memory DESC LIMIT 10\G``` |
| 临时表使用统计            | `performance_schema.session_status`     | ```SHOW GLOBAL STATUS LIKE 'Created_tmp%';```                          |

#### InnoDB 引擎
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 缓冲池命中率              | `performance_schema.global_status`      | ```SELECT VARIABLE_VALUE FROM global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads';```<br>计算：`(1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests) * 100` |
| 脏页比例                  | `information_schema.INNODB_BUFFER_PAGE` | ```SELECT COUNT(*)/@@innodb_buffer_pool_size*100 FROM INNODB_BUFFER_PAGE WHERE OLDEST_MODIFICATION > 0\G``` |
| 刷盘负载                  | `performance_schema.global_status`      | ```SELECT VARIABLE_NAME, VARIABLE_VALUE FROM global_status WHERE VARIABLE_NAME LIKE 'Innodb_buffer_pool_wait%'\G``` |
| IO压力指标                | `information_schema.INNODB_METRICS`     | ```SELECT NAME, COUNT FROM INNODB_METRICS WHERE NAME LIKE 'buffer_pool_wait%'\G``` |

---

### 7.4 关键配置验证
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 事务隔离级别全局设置      | `performance_schema.global_variables`   | ```SELECT * FROM global_variables WHERE VARIABLE_NAME IN ('transaction_isolation','tx_isolation')\G``` |
| Redo Log 配置状态         | `performance_schema.global_variables`   | ```SELECT * FROM global_variables WHERE VARIABLE_NAME LIKE 'innodb_log_file%'\G``` |
| 自动提交状态              | `performance_schema.variables_by_thread`| ```SELECT * FROM variables_by_thread WHERE VARIABLE_NAME = 'autocommit'\G``` |

---

### 7.5 容量监控
| 查询内容                  | 查询表/视图                             | 查询 SQL                                                                 |
|---------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| 表空间使用率              | `information_schema.FILES`              | ```SELECT FILE_NAME, TOTAL_EXTENTS * EXTENT_SIZE / 1024 / 1024 AS "Size(MB)" FROM FILES WHERE FILE_TYPE = 'TABLESPACE'\G``` |
| 索引碎片率                | `information_schema.TABLES`             | ```SELECT TABLE_NAME, DATA_FREE / 1024 / 1024 AS "Frag(MB)" FROM TABLES WHERE DATA_FREE > 10000000\G``` |
| 大表TOP10                 | `information_schema.TABLES`             | ```SELECT TABLE_SCHEMA, TABLE_NAME, DATA_LENGTH/1024/1024 AS "Size(MB)" FROM TABLES ORDER BY DATA_LENGTH DESC LIMIT 10\G``` |

---

**使用说明：**
1. `\G` 替换为 `;` 可表格化显示（适用命令行）
2. MySQL 8.0+ 优先使用 `performance_schema` 和 `sys` 库
3. 关键指标建议定期采集（如每分钟采集 `SHOW GLOBAL STATUS` 的快照）
4. 生产环境使用 `sys` 库视图更友好：`sys.innodb_lock_waits`, `sys.schema_table_lock_waits` 等

