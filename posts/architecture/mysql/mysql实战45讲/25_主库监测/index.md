# 19 MySQL 主库监测


怎么判断一个主库是否出了问题？

<!-- more -->

## 1. 主库监测
在一主一备的双 M 架构里，主备切换只需要把客户端流量切到备库；而在一主多从架构里，主备切换除了要把客户端流量切到备库外，还需要把从库接到新主库上。

主备切换有两种场景，一种是主动切换，一种是被动切换。而其中被动切换，往往是因为主库出问题了，由 HA 系统发起的。那怎么判断主库是否出了问题，有以下几种方案:
1. select 1 判断

### 1.1 select 1 判断
select 1 成功返回，只能说明这个库的进程还在，并不能说明主库没问题。

innodb_thread_concurrency 参数的目的是，控制 InnoDB 的并发线程上限。也就是说，一旦并发线程数达到这个值，InnoDB 在接收到新请求的时候，就会进入等待状态，直到有线程退出。此时select 1 是能执行成功的，但是查询表 t 的语句会被堵住，系统已经处于不可用状态。

#### innodb_thread_concurrency
innodb_thread_concurrency 这个参数的默认值是 0，表示不限制并发线程数量。但是，不限制并发线程数肯定是不行的。因为，一个机器的 CPU 核数有限，线程全冲进来，上下文切换的成本就会太高。

通常情况下，我们建议把 innodb_thread_concurrency 设置为 64~128 之间的值。

需要注意的是
1. 线程处于空闲状态，不算在并发线程里面。
2. 在线程进入锁等待以后，也就是说等行锁（也包括间隙锁）的线程也不算在并发线程内
3. 对于真正执行的查询，比如 `select sleep(100) from t`，还是要算进并发线程内

#### 并发连接与并发查询
show processlist 的结果指的就是并发连接。而“当前正在执行”的语句，才是我们所说的并发查询。
1. 并发连接数达到几千个影响并不大，就是多占一些内存而已
2. 并发查询太高才是 CPU 杀手。这也是为什么我们需要设置 innodb_thread_concurrency 参数的原因


### 1.2 更新判断
为了能够检测 InnoDB 并发线程数过多导致的系统不可用情况，我们需要找一个访问 InnoDB 的场景。

同时更新事务要写 binlog，而一旦 binlog 所在磁盘的空间占用率达到 100%，那么所有的更新语句和事务提交的 commit 语句就都会被堵住。但是，系统这时候还是可以正常读数据的。因此我们应该使用更新语句，而不是查询语句作为监控语句。更新语句类似于:

```bash
mysql> update mysql.health_check set t_modified=now();
```
 timestamp 字段，用来表示最后一次执行检测的时间。

对于双主模型，如果主库 A 和备库 B 都用相同的更新命令进行监测，就可能出现行冲突。我们可以在 mysql.health_check 表上存入多行数据，并用 A、B 的 server_id 做主键: 

```bash
mysql> CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```

更新判断是一个相对比较常用的方案了，不过依然存在一些问题。其中，“判定慢”一直是让 DBA 头疼的问题。这里涉及到的是服务器 IO 资源分配的问题。检测使用的 update 命令，需要的IO资源很少，大概率可以执行成功。但可能此时系统的 IO 利用率已经达到 100%，正常的 SQL 已经执行的很慢了。

根本原因是我们上面说的所有方法，都是基于外部检测的。外部检测天然有一个问题，就是随机性。因为，外部检测都需要定时轮询，所以系统可能已经出问题了，但是却需要等到下一个检测发起执行语句的时候，我们才有可能发现问题。

### 1.3 内部统计
MySQL 5.6 版本以后提供的 performance_schema 库，就在 file_summary_by_event_name 表里统计了每次 IO 请求的时间。

```bash
MariaDB [performance_schema]> select * from file_summary_by_event_name where event_name like "%innodb_log%" \G
*************************** 1. row ***************************
               EVENT_NAME: wait/io/file/innodb/innodb_log_file   # 统计的是 redo log 的写入时间
               COUNT_STAR: 94                    # 所有 IO 的总次数
           SUM_TIMER_WAIT: 247391627463          # 所有 IO 类型的统计，单位是皮秒
           MIN_TIMER_WAIT: 0
           AVG_TIMER_WAIT: 2631825709
           MAX_TIMER_WAIT: 11638807401
               
               COUNT_READ: 6                     # 读操作的统计     
           SUM_TIMER_READ: 30000390
           MIN_TIMER_READ: 0
           AVG_TIMER_READ: 5000065
           MAX_TIMER_READ: 13752883
 SUM_NUMBER_OF_BYTES_READ: 68096                 # 总共从 redo log 统计了多少字节

              COUNT_WRITE: 43                    # 写操作统计
          SUM_TIMER_WRITE: 1621529472
          MIN_TIMER_WRITE: 0
          AVG_TIMER_WRITE: 37709927
          MAX_TIMER_WRITE: 73538815
SUM_NUMBER_OF_BYTES_WRITE: 24576

               COUNT_MISC: 45                    # 其他类型的统计，对于redolog 就是 fsync 
           SUM_TIMER_MISC: 245740097601
           MIN_TIMER_MISC: 0
           AVG_TIMER_MISC: 5460890834
           MAX_TIMER_MISC: 11638807401
1 row in set (0.001 sec)
```


#### 启用 performance_schema
打开所有的 performance_schema 项，性能大概会下降 10% 左右。所以，我建议你只打开自己需要的项进行统计。你可以通过下面的方法打开或者关闭某个具体项的统计。

```bash
# 打开 redo log 的统计项
mysql> update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';
```

#### 异常检测
比如，你可以设定阈值，单次 IO 请求时间超过 200 毫秒属于异常，然后使用类似下面这条语句作为检测逻辑。
```bash
mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;

# 清空统计数据，以免对下次检测产生影响
mysql> truncate table performance_schema.file_summary_by_event_name;
```

### 1.4 方案总结
建议是优先考虑 update 系统表，然后再配合增加检测 performance_schema 的信息。

