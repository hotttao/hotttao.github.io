# 20 MySQL 误删恢复


误删数据恢复

<!-- more -->

## 1. 误删数据
传统的高可用架构是不能预防误删数据的，因为主库的一个 drop table 命令，会通过 binlog 传给所有从库和级联从库，进而导致整个集群的实例都会执行这个命令。

为了找到解决误删数据的更高效的方法，我们需要先对和 MySQL 相关的误删数据，做下分类：

1. 使用 delete 语句误删数据行；
2. 使用 drop table 或者 truncate table 语句误删数据表；
3. 使用 drop database 语句误删数据库；
4. 使用 rm 命令误删整个 MySQL 实例

恢复数据比较安全的做法，是恢复出一个备份，或者找一个从库作为临时库，在这个临时库上执行这些操作，然后再将确认过的临时库的数据，恢复回主库。

这是因为，一个在执行线上逻辑的主库，数据状态的变更往往是有关联的。可能由于发现数据问题的时间晚了一点儿，就导致已经在之前误操作的基础上，业务代码逻辑又继续修改了其他数据。所以，如果这时候单独恢复这几行数据，而又未经确认的话，就可能会出现对数据的二次破坏。

### 1.1 误删预防
误删更重要的是事前预防:
1. 把 sql_safe_updates 参数设置为 on。这样一来，如果我们忘记在 delete 或者 update 语句中写 where 条件，或者 where 条件里面没有包含索引字段的话，这条语句的执行就会报错。
2. 码上线前，必须经过 SQL 审计。
3. 账号分离，避免写错命令。
4. 第二条建议是，制定操作规范。这样做的目的，是避免写错要删除的表名
	- 在删除数据表之前，必须先对表做改名操作。然后，观察一段时间，确保对业务无影响以后再删除这张表。
	- 改表名的时候，要求给表名加固定的后缀（比如加 \_to_be_deleted)，然后删除表的动作必须通过管理系统执行。并且，管理系删除表的时候，只能删除固定后缀的表。
5. 脚本分别是：备份脚本、执行脚本、验证脚本和回滚脚本。如果能够坚持做到，即使出现问题，也是可以很快恢复的，一定能降低出现故障的概率。

## 2. 误删行
使用 delete 语句误删了数据行，可以用 Flashback 工具通过闪回把数据恢复回来。而能够使用这个方案的前提是，需要确保 binlog_format=row 和 binlog_row_image=FULL。

Flashback 恢复数据的原理，是修改 binlog 的内容，拿回原库重放。
1. 对于 insert 语句，对应的 binlog event 类型是 Write_rows event，把它改成 Delete_rows event 即可；
2. 对于 delete 语句，也是将 Delete_rows event 改为 Write_rows event；
3. 而如果是 Update_rows 的话，binlog 里面记录了数据行修改前和修改后的值，对调这两行的位置即可。

但是，delete 全表是很慢的，需要生成回滚日志、写 redo、写 binlog。所以，从性能角度考虑，你应该优先考虑使用 truncate table 或者 drop table 命令。

## 3. 误删库 / 表
使用 truncate /drop table 和 drop database 命令删除的数据，就没办法通过 Flashback 来恢复了。因为即使我们配置了 binlog_format=row，执行这三个命令时，记录的 binlog 还是 statement 格式。binlog 里面就只有一个 truncate/drop 语句，这些信息是恢复不出数据的。

这个时候就需要使用全量备份，加增量日志了，这个方案要求线上有定期的全量备份，并且实时备份 binlog。

恢复数据的流程如下:
1. 取最近一次全量备份，假设这个库是一天一备，上次备份是当天 0 点；
2. 用备份恢复出一个临时库；
3. 从日志备份里面，取出凌晨 0 点之后的日志；
4. 把这些日志，除了误删除数据的语句外，全部应用到临时库

![recover_table](/images/mysql/MySQL45讲/recover_table.png)

关于这个过程，需要说明的是:
1. mysqlbinlog 有一个–database 参数，用来指定误删表所在的库，这样可以跳过其他库，加快恢复速度
2. 需要跳过 1 2 点误操作的语句:
	- 使用了 GTID 模式: 假设误操作命令的 GTID 是 gtid1，那么只需要执行 `set gtid_next=gtid1;begin;commit;` 就可以跳过误删的操作
	- 未使用 GTID 模式: 手动使用 --start-position --stop-position 跳过误删的操作
3. 这个恢复的操作还是不够快，原因有以下两点:
	- 如果是误删表，最好就是只恢复出这张表，也就是只重放这张表的操作，但是 mysqlbinlog 工具并不能指定只解析一个表的日志；
	- 用 mysqlbinlog 解析出日志应用，应用日志的过程就只能是单线程。

### 3.1 使用备库同步方式恢复数据
更快的方法是在用备份恢复出临时实例之后，将这个临时实例设置成线上备库的从库，这样：
1. 在 start slave 之前，先通过执行﻿
`﻿change replication filter replicate_do_table = (tbl_name)` 命令，就可以让临时库只同步误操作的表；
2. 这样做可以用上并行复制技术，来加速整个数据恢复过程。

![recover_slave](/images/mysql/MySQL45讲/recover_slave.png)

关于这个过程需要说明的是:
1. binlog 备份系统到线上备库有一条虚线，是指如果由于时间太久，备库上已经删除了临时实例需要的 binlog 的话，我们可以从 binlog 备份系统中找到需要的 binlog，再放回备库中。
2. 同步过程，同样需要跳过误删的操作

#### 删掉的 binlog 放回备库的操作步骤
1. 从备份系统下载 master.000005 和 master.000006 这两个文件，放到备库的日志目录下；
2. 打开日志目录下的 master.index 文件，在文件开头加入两行，内容分别是 “./master.000005”和“./master.000006”;
3. 重启备库，目的是要让备库重新识别这两个日志文件；

### 3.2 延迟复制备库
上面利用备库并行复制的方案仍然存在恢复时间不可控问题。如果一个库的备份特别大，或者误操作的时间距离上一个全量备份的时间较长，这个恢复时间可能是要按天来计算的。

我们可以考虑搭建延迟复制的备库。这个功能是 MySQL 5.6 版本引入的。

一般的主备复制结构存在的问题是，如果主库上有个表被误删了，这个命令很快也会被发给所有从库，进而导致所有从库的数据表也都一起被误删了。

延迟复制的备库是一种特殊的备库，通过 `CHANGE MASTER TO MASTER_DELAY = N`命令，可以指定这个备库持续保持跟主库有 N 秒的延迟。只要在延迟的时间内发现误删，这个命令就还没有在这个延迟复制的备库执行。这时候到这个备库上执行 stop slave，再通过之前介绍的方法，跳过误操作命令，就可以恢复出需要的数据。


## 4. rm 删除数据
对于一个有高可用机制的 MySQL 集群来说，最不怕的就是 rm 删除数据了。只要不是恶意地把整个集群删除，而只是删掉了其中某一个节点的数据的话，HA 系统就会开始工作，选出一个新的主库，从而保证整个集群的正常工作。
