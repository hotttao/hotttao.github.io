# 15 MySQL 主从复制


接下来内容与 MySQL 主从复制相关的内容，包括:
1. MySQL 如何保证主从同步的最终一致性，设计 binlog 同步过程和binlog 的格式
2. 主备延迟，以及并行复制
3. 由于主备延迟不可避免，因此由两种主备切换策略
4. 主从复制同步位点的判断以及GTID 模式
5. 读写分离，以及怎么处理主备延迟导致的读写分离问题
6. 主库的心跳信息监测，以便在主库故障时及时进行主备切换
<!-- more -->


## 1. 主备的基本原理
如图 1 所示就是基本的主备切换流程

![cluster_change](/images/mysql/MySQL45讲/cluster_change.png)

在状态 1 中，客户端的读写都直接访问节点 A，而节点 B 是 A 的备库，只是将 A 的更新都同步过来，到本地执行。这样可以保持节点 B 和 A 的数据是相同的。当需要切换的时候，就切成状态 2。

在状态 1 中，虽然节点 B 没有被直接访问，但是我依然建议你把节点 B（也就是备库）设置成只读（readonly）模式。这样做，有以下几个考虑：
1. 有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作；
2. 防止切换逻辑有 bug，比如切换过程中出现双写，造成主备不一致；
3. 可以用 readonly 状态，来判断节点的角色

readonly 设置对超级 (super) 权限用户是无效的，而用于同步更新的线程，拥有超级权限，因此 readonly 不会影响主从同步的写。

## 2. binlog 同步过程
![binlog_thread](/images/mysql/MySQL45讲/binlog_thread.png)

图中画出的就是一个 update 语句在节点 A 执行，然后同步到节点 B 的完整流程图。可以看到：主库接收到客户端的更新请求后，执行内部事务的更新逻辑，同时写 binlog。

一个事务日志同步的完整过程是这样的：
1. 在备库 B 上通过 change master 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给 B
4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）
5. sql_thread 读取中转日志，解析出日志里的命令，并执行。

后来由于多线程复制方案的引入，sql_thread 演化成为了多个线程，这个我们后面在详述。

## 3. binlog 的格式
binlog 有两种格式，一种是 statement，一种是 row。第三种格式 mixed 是前两种格式的混合。

### 3.1 binlog 查看方法
通过如下的方法我们可以查看每种binlog 格式记录的详细内容:

```bash
# 1. 查看当前系统的二进制文件
MariaDB [(none)]> SHOW {BINARY | MASTER} LOGS
+-------------------+-----------+
| Log_name          | File_size |
+-------------------+-----------+
| master-log.000001 |      1254 |
| master-log.000002 |       435 |
| master-log.000003 |       435 |
| master-log.000004 |      2389 |

# 2. 查看 binlog 日志
MariaDB [(none)]> SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count]
MariaDB [(none)]> show binlog events in 'master-log.000004';
MariaDB [(none)]> show binlog events in 'master-log.000004' from 387 limit 3;
+-------------------+-----+------------+-----------+-------------+--------------------------------------------------------------+
| Log_name          | Pos | Event_type | Server_id | End_log_pos | Info                                                         |
+-------------------+-----+------------+-----------+-------------+--------------------------------------------------------------+
| master-log.000004 | 387 | Gtid       |         1 |         429 | BEGIN GTID 0-1-6                                             |
| master-log.000004 | 429 | Query      |         1 |         544 | use `tsong`; insert into course values(2, "计算机网络")      |
| master-log.000004 | 544 | Xid        |         1 |         575 | COMMIT /* xid=205 */                                         |
+-------------------+-----+------------+-----------+-------------+--------------------------------------------------------------+
3 rows in set (0.000 sec)

# 3. 通过 mysqlbinlog 解析 binlog 的具体内容
 mysqlbinlog  -vv master-log.000004 --start-position=387 --stop-position=390;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#200311 12:25:09 server id 1  end_log_pos 256 CRC32 0xe5385d26  Start: binlog v 4, server v 10.4.12-MariaDB-log created 200311 12:25:09
BINLOG '
JWhoXg8BAAAA/AAAAAABAAAAAAQAMTAuNC4xMi1NYXJpYURCLWxvZwAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAEzgNAAgAEgAEBAQEEgAA5AAEGggAAAAICAgCAAAACgoKAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAEEwQADQgICAoKCgEmXTjl
'/*!*/;
# at 387
#200311 12:27:27 server id 1  end_log_pos 429 CRC32 0x75f470c3  GTID 0-1-6 trans
/*!100101 SET @@session.skip_parallel_replication=0*//*!*/;
/*!100001 SET @@session.gtid_domain_id=0*//*!*/;
/*!100001 SET @@session.server_id=1*//*!*/;
/*!100001 SET @@session.gtid_seq_no=6*//*!*/;
BEGIN
/*!*/;
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

### 3.2 试验环境
为了便于描述 binlog 的这三种格式间的区别，我们使用如下的环境，并执行最后的 delete 语句
```bash

mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');

# 用 MySQL 客户端来做这个实验的话，要记得加 -c 参数，否则客户端会自动去掉注释。
mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;
```

### 3.3 statement
binlog_statement
当 binlog_format=statement 时，`show binlog events in 'master.000001';` binlog 内容如下:

![binlog_statement](/images/mysql/MySQL45讲/binlog_statement.png)

1. 第一行 SET @@SESSION.GTID_NEXT='ANONYMOUS’与主备切换有关，后面我们在详述
2. 第二行是一个 BEGIN，跟第四行的 commit 对应，表示中间是一个事务；
3. 第三行就是真实执行的语句了，statement 格式下，记录到 binlog 里的是语句原文
4. 最后一行是一个 COMMIT，里面写着 xid=61。xid 是 binlog 和 redo log 的关联字段，同时也是 statement 格式binlog 的完整性标识。

因为 statement 记录的原始语句，而不是具体的行，当使用 delete limit 等情况时，就有可能因为主备执行逻辑(比如执行时选择的索引)不同出现主备不一致的情况。


### 3.4 row 格式
当 binlog_format=row 时，`show binlog events in 'master.000001';`binlog 内容如下:

![binlog_row](/images/mysql/MySQL45讲/binlog_raw.png)

row 格式的 binlog 里没有了 SQL 语句的原文，而是替换成了两个 event：Table_map 和 Delete_rows：
1. Table_map event，用于说明接下来要操作的表是 test 库的表 t;
2. Delete_rows event，用于定义删除的行为。

show binlog 是无法查看 row binlog 的详细内容的，需要借助 mysqlbinlog 工具。下面是使用 mysqlbinlog 查看的 row 格式的 binlog 内容:

```bash 
mysqlbinlog  -vv data/master.000001 --start-position=8900;
```

![binlog_content](/images/mysql/MySQL45讲/binlog_content.png)

row 格式的 binlog 包括以下内容:
1. server id 1，表示这个事务是在 server_id=1 的这个库上执行的。
2. 每个 event 都有 CRC32 的值，这是参数 binlog_checksum 设置成了 CRC32，用于检查 binlog 是否完整
3. binlog 记录的每个 SQL 都会被翻译成多个 event:
	- Table_map event 显示了接下来要打开的表，map 到数字 226。如果要操作多张表呢？每个表都有一个对应的 Table_map event、都会 map 到一个单独的数字，用于区分对不同表的操作。
	- Delete_rows event，用于定义删除的行为
4.  mysqlbinlog 的命令中，使用了 -vv 参数是为了把内容都解析出来，所以从结果里面可以看到各个字段的值（比如，@1=4、 @2=4 这些值）
5. binlog_row_image 的默认配置是 FULL，因此 Delete_event 里面，包含了删掉的行的所有字段的值。如果把 binlog_row_image 设置为 MINIMAL，则只会记录必要的信息，在这个例子里，就是只会记录 id=4 这个信息。
6. 最后的 Xid event，用于表示事务被正确地提交了。

当 binlog_format 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题。

## 3.4 mixed 格式
为什么会有 mixed 这种 binlog 格式的存在场景？
1. 因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
2. 但 row 格式的缺点是，很占空间。比如你用一个 delete 语句删掉 10 万行数据，用 statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中。这样做，不仅会占用更大的空间，同时写 binlog 也要耗费 IO 资源，影响执行速度。
3. 所以，MySQL 就取了个折中方案，也就是有了 mixed 格式的 binlog。mixed 格式的意思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。

也就是说，mixed 格式可以利用 statment 格式的优点，同时又避免了数据不一致的风险。

### 3.5 数据恢复
但是 MySQL 的 binlog 格式最好还是设置成 row。因为需要 binlog 做数据恢复。
1. delete 语句，row 格式的 binlog 也会把被删掉的行的整行信息保存起来
2. insert 语句的 binlog 里会记录所有的字段信息，这些信息可以用来精确定位刚刚被插入的那一行
3. update 语句，binlog 里面会记录修改前整行的数据和修改后的整行数据。

MariaDB 的Flashback工具正是基于 row 格式的 binlog 做数据恢复的工具。

如果直接使用 binlog 做数据恢复，标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。类似下面的命令：

```bash
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```
记住，不要直接去binlog 内复制语句去执行，因为 mysql 会自动根据 sql 执行的上下文添加类似库切换，设置时间戳等命令，这些语句是需要同时被执行的。

## 4. 循环复制问题
实际生产上使用比较多的是双 M 结构，也就是下图所示的主备切换流程。

![cluster_two](/images/mysql/MySQL45讲/cluster_two.png)

双主模型中存在循环复制的问题: 业务逻辑在节点 A 上更新了一条语句，然后再把生成的 binlog 发给节点 B，节点 B 执行完这条更新语句后也会生成 binlog。（建议把参数 log_slave_updates 设置为 on，表示备库执行 relay log 后生成 binlog，这样任意主库上的 binlog 都可以用作数据恢复）

那么，如果节点 A 同时是节点 B 的备库，相当于又把节点 B 新生成的 binlog 拿过来执行了一次，然后节点 A 和 B 间，会不断地循环执行这个更新语句，也就是循环复制了。这个要怎么解决呢？

MySQL 是这样解决循环复制的问题的:
1. 两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
2. 从节点 A 更新的事务，binlog 里面记的都是 A 的 server id；
3. 传到节点 B 执行一次以后，节点 B 生成的 binlog 的 server id 也是 A 的 server id；
4. 再传回给节点 A，A 判断到这个 server id 与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了

按照这个逻辑，如果我们设置了双 M 结构，日志的执行流就会变成这样：
1. 从节点 A 更新的事务，binlog 里面记的都是 A 的 server id；
2. 传到节点 B 执行一次以后，节点 B 生成的 binlog 的 server id 也是 A 的 server id；
3. 再传回给节点 A，A 判断到这个 server id 与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了。

但是即便如此存在下面两种情况一样会导致循环复制问题。

### 4.1 场景一
在一个主库更新事务后，用命令 set global server_id=x 修改了 server_id。等日志再传回来的时候，发现 server_id 跟自己的 server_id 不同，就只能执行了。

### 4.2 场景二
有三个节点的时候，如图 7 所示，trx1 是在节点 B 执行的，因此 binlog 上的 server_id 就是 B，binlog 传给节点 A，然后 A 和 A’搭建了双 M 结构，就会出现循环复制。

这种三节点复制的场景，做数据库迁移的时候会出现。可以在 A 或者 A’上，执行如下命令：

```bash
stop slave；
CHANGE MASTER TO IGNORE_SERVER_IDS=(server_id_of_B);
start slave;
```

这样这个节点收到日志后就不会再执行。过一段时间后，再执行下面的命令把这个值改回来。

```bash
stop slave；
CHANGE MASTER TO IGNORE_SERVER_IDS=();
start slave;
```
