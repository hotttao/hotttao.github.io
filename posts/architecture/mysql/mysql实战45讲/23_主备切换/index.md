# 17 MySQL 主备切换


MySQL 主备切换策略，一主多从
<!-- more -->


## 1. 双主模型的主备切换策略
如图 1 所示就是基本的主备切换流程

![cluster_change](/images/mysql/MySQL45讲/cluster_change.png)

正常情况下，只要主库执行更新生成的所有 binlog，都可以传到备库并被正确地执行，备库就能达到跟主库一致的状态，这就是最终一致性。但是，MySQL 要提供高可用能力，只有最终一致性是不够的。由于主备延迟的存在，所以在主备切换的时候，就相应的有不同的策略。

### 1.1 可靠性优先策略

![stable_first](/images/mysql/MySQL45讲/stable_first.png)

如上图，可靠性优先的主备切换流程:
1. 判断备库 B 现在的 seconds_behind_master，如果小于某个值（比如 5 秒）继续下一步，否则持续重试这一步；
2. 把主库 A 改成只读状态，即把 readonly 设置为 true；
3. 判断备库 B 的 seconds_behind_master 的值，直到这个值变成 0 为止；
4. 把备库 B 改成可读写状态，也就是把 readonly 设置为 false；把业务请求切到备库 B

以看到，这个切换流程中是有不可用时间的。因为在步骤 2 之后，主库 A 和备库 B 都处于 readonly 状态，也就是说这时系统处于不可写状态，直到步骤 5 完成后才能恢复。

假设，主库 A 和备库 B 间的主备延迟是 30 分钟，这时候主库 A 掉电了。此时我们必须等待备库执行完中转日志，才能切换到备库 B。这段时间，系统处于完全不可用的状态。所以在满足数据可靠性的前提下，MySQL 高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。

### 1.2 可用性优先
如果强行把步骤 4、5 调整到最开始执行，也就是说不等主备数据同步，那么系统几乎就没有不可用时间了。代价就是可能出现数据不一致的情况。使用 row 格式的 binlog 时，数据不一致问题会导致 mysql 报错，更容易被发现。而使用 mixed 或者 statement 格式的 binlog 时，数据很可能悄悄地就不一致了。如果你过了很久才发现数据不一致的问题，很可能这时的数据不一致已经不可查，或者连带造成了更多的数据逻辑不一致。

大多数情况下，应该使用可靠性优先策略。毕竟对数据服务来说的话，数据的可靠性一般还是要优于可用性的。在这个基础上，通过减少主备延迟，提升系统的可用性。

在满足数据可靠性的前提下，MySQL 高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。

## 2. 一主多从
大多数的互联网应用场景都是读多写少，因此你负责的业务，在发展过程中很可能先会遇到读性能的问题。此时我们就需要 MySQL 一主多从架构。我们将分成两个方面来讲解一主多从:
1. 一主多从的切换正确性
2. 一主多从的查询逻辑正确性的方法: 见下一节"读写分离"

下面，就是一个基本的一主多从结构:
1. 虚线箭头表示的是主备关系，也就是 A 和 A’互为主备
2. 从库 B、C、D 指向的是主库 A。
3. 一主多从的设置，一般用于读写分离，主库负责所有的写入和一部分读，其他的读请求则由从库分担。

![multi_slave](/images/mysql/MySQL45讲/multi_slave.png)

## 3. 一主多从的主备切换
如图 2 所示，就是主库发生故障，主备切换后的结果。

![slave_change](/images/mysql/MySQL45讲/slave_change.png)

相比于一主一备的切换流程，一主多从结构在切换完成后，A’会成为新的主库，从库 B、C、D 也要改接到 A’。

### 3.1 基于位点的主备切换
当我们把节点 B 设置成节点 A’的从库的时候，需要执行一条 change master 命令:
```bash
CHANGE MASTER TO 
MASTER_HOST=$host_name 
MASTER_PORT=$port 
MASTER_USER=$user_name 
MASTER_PASSWORD=$password 
MASTER_LOG_FILE=$master_log_name 
MASTER_LOG_POS=$master_log_pos  
```

最后两个参数 MASTER_LOG_FILE 和 MASTER_LOG_POS 表示，要从主库的 master_log_name 文件的 master_log_pos 这个位置的日志继续同步。而这个位置就是我们所说的同步位点，也就是主库对应的文件名和日志偏移量。

原来节点 B 是 A 的从库，本地记录的也是 A 的位点。但是相同的日志，A 的位点和 A’的位点是不同的。因此，从库 B 要切换的时候，就需要先经过“找同步位点”这个逻辑。这个位点很难精确取到。


一种取同步位点的方法是这样的：
1. 等待新主库 A’把中转日志（relay log）全部同步完成；
2. 在 A’上执行 show master status 命令，得到当前 A’上最新的 File 和 Position；
3. 取原主库 A 故障的时刻 T；
3. 用 mysqlbinlog 工具解析 A’的 File，得到 T 时刻的位点。

```bash
mysqlbinlog File --stop-datetime=T --start-datetime=T
```

网络延迟的不确定性，从节点 B 是否已经执行过T时刻的位点是不确定的，因此我们从时刻 T 的位点同步时就有可能出现主键冲突(insert 语句被重复执行)。

通常情况下，我们在切换任务的时候，要先主动跳过这些错误，有两种常用的方法。
```bash
# 方法一: 手动跳过一个事务
# 从库 B 刚开始接到新主库 A’时，持续观察，每次碰到这些错误就停下来，执行一次跳过命令
set global sql_slave_skip_counter=1;
start slave;

# 方法二: 通过设置 slave_skip_errors 参数，直接设置跳过指定的错误。
```

在执行主备切换时，有这么两类错误，是经常会遇到的：
1. 1062 错误是插入数据时唯一键冲突；
2. 1032 错误是删除数据时找不到行。

我们可以把 slave_skip_errors 设置为 “1032,1062”，这样中间碰到这两个错误时就直接跳过。

这个背景是，我们很清楚在主备切换过程中，直接跳过 1032 和 1062 这两类错误是无损的，所以才可以这么设置 slave_skip_errors 参数。等到主备间的同步关系建立完成，并稳定执行一段时间之后，我们还需要把这个参数设置为空，以免之后真的出现了主从数据不一致，也跳过了。

### 3.2 GTID
基于位点的主备切换，复杂也容易出错，所以，MySQL 5.6 版本引入了 GTID，彻底解决了这个困难。

GTID 的全称是 Global Transaction Identifier，也就是全局事务 ID，是一个事务在提交的时候生成的，是这个事务的唯一标识。它由两部分组成，格式是：`GTID=server_uuid:gno`
1. server_uuid 是一个实例第一次启动时自动生成的，是一个全局唯一的值；
2. gno 是一个整数，初始值是 1，每次提交事务的时候分配给这个事务，并加 1。

注意: 事务 id 是在事务执行过程中分配的，如果这个事务回滚了，事务 id 也会递增，而 gno 是在事务提交的时候才会分配。

#### GTID 的使用
GTID 模式的启动也很简单，只需配置两个参数:
1. gtid_mode=on
2. enforce_gtid_consistency=on

在 GTID 模式下，每个事务都会跟一个 GTID 一一对应。这个 GTID 有两种生成方式，而使用哪种方式取决于 session 变量 gtid_next 的值。
1. `gtid_next=automatic`
2. `gtid_next 是一个指定的 GTID 的值`

**`gtid_next=automatic`** 代表使用默认值。这时，MySQL 就会把 server_uuid:gno 分配给这个事务。
1. 记录 binlog 的时候，先记录一行 SET @@SESSION.GTID_NEXT=‘server_uuid:gno’;
2. 把这个 GTID 加入本实例的 GTID 集合

如果 gtid_next 是一个指定的 GTID 的值，比如通过 set gtid_next='current_gtid’指定为 current_gtid，那么就有两种可能：
1. 如果 current_gtid 已经存在于实例的 GTID 集合中，接下来执行的这个事务会直接被系统忽略；
2. 如果 current_gtid 没有存在于实例的 GTID 集合中，就将这个 current_gtid 分配给接下来要执行的事务，也就是说系统不需要给这个事务生成新的 GTID，因此 gno 也不用加 1

一个 current_gtid 只能给一个事务使用。这个事务提交后，如果要执行下一个事务，就要执行 set 命令，把 gtid_next 设置成另外一个 gtid 或者 automatic。这样，每个 MySQL 实例都维护了一个 GTID 集合，用来对应“这个实例执行过的所有事务”。

通过提交一个特定 GTID 的空事务，我们就可以实现跳过主服务器同步过来的特定时事务:

```bash
# 把这个 GTID 加到从库 的 GTID 集合中
set gtid_next='aaaaaaaa-cccc-dddd-eeee-ffffffffffff:10';
begin;
commit;

# 恢复 GTID 的默认分配行为
set gtid_next=automatic;
start slave;
```

## 4. 基于 GTID 的主备切换
在 GTID 模式下，备库 B 要设置为新主库 A’的从库的语法如下：
```bash
CHANGE MASTER TO 
MASTER_HOST=$host_name 
MASTER_PORT=$port 
MASTER_USER=$user_name 
MASTER_PASSWORD=$password 
master_auto_position=1 
```

master_auto_position=1 就表示这个主备关系使用的是 GTID 协议。我们把现在这个时刻，实例 A’的 GTID 集合记为 set_a，实例 B 的 GTID 集合记为 set_b。接下来，我们就看看现在的主备切换逻辑。

我们在实例 B 上执行 start slave 命令，取 binlog 的逻辑是这样的：
1. 实例 B 指定主库 A’，基于主备协议建立连接。
2. 实例 B 把 set_b 发给主库 A’。
3. 实例 A’算出所有存在于 set_a，但是不存在于 set_b 的 GTID 的集合，判断 A’本地是否包含了这个差集需要的所有 binlog 事务。
	- 如果不包含，表示 A’已经把实例 B 需要的 binlog 给删掉了，直接返回错误；
	- 如果确认全部包含，A’从自己的 binlog 文件里面，找出第一个不在 set_b 的事务，发给 B；
4. 之后就从这个事务开始，往后读文件，按顺序取 binlog 发给 B 去执行

在基于 GTID 的主备关系里，系统认为只要建立主备关系，就必须保证主库发给备库的日志是完整的。跟基于位点的主备协议不同。基于位点的协议，是由备库决定的，备库指定哪个位点，主库就发哪个位点，不做日志的完整性判断。


之后这个系统就由新主库 A’写入，主库 A’的自己生成的 binlog 中的 GTID 集合格式是：server_uuid_of_A’:1-M。如果之前从库 B 的 GTID 集合格式是 server_uuid_of_A:1-N， 那么切换之后 GTID 集合的格式就变成了 server_uuid_of_A:1-N, server_uuid_of_A’:1-M。当然，主库 A’之前也是 A 的备库，因此主库 A’和从库 B 的 GTID 集合是一样的。这就达到了我们预期。


### 4.1 问题
在 GTID 模式下，如果一个新的从库接上主库，但是需要的 binlog 已经没了，要怎么做？
1. 如果业务允许主从不一致的情况，那么可以在主库上先执行 show global variables like ‘gtid_purged’，得到主库已经删除的 GTID 集合，假设是 gtid_purged1；然后先在从库上执行 reset master，再执行 set global gtid_purged =‘gtid_purged1’；最后执行 start slave，就会从主库现存的 binlog 开始同步。binlog 缺失的那一部分，数据在从库上就可能会有丢失，造成主从不一致。
2. 如果需要主从数据一致的话，最好还是通过重新搭建从库来做。
3. 如果有其他的从库保留有全量的 binlog 的话，可以把新的从库先接到这个保留了全量 binlog 的从库，追上日志以后，如果有需要，再接回主库。
4. 如果 binlog 有备份的情况，可以先在从库上应用缺失的 binlog，然后再执行 start slave。

## 5. GTID 和在线 DDL
假设，这两个互为主备关系的库还是实例 X 和实例 Y，且当前主库是 X，并且都打开了 GTID 模式。这时的主备切换流程可以变成下面这样：

1. 在实例 X 上执行 stop slave。在实例 Y 上执行 DDL 语句。
2. 注意，这里并不需要关闭 binlog。
3. 执行完成后，查出这个 DDL 语句对应的 GTID，并记为 server_uuid_of_Y:gno。
4. 到实例 X 上执行以下语句序列：
```bash

set GTID_NEXT="server_uuid_of_Y:gno";
begin;
commit;
set gtid_next=automatic;
start slave;
```

这样做的目的在于，既可以让实例 Y 的更新有 binlog 记录，同时也可以确保不会在实例 X 上执行这条更新。

