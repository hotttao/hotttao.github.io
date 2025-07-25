# 4 MYSQL 锁


全局锁 - 表锁 - 行锁

<!-- more -->

## 1. 全局锁
全局锁:
- 作用: 对整个数据库实例加锁
- 加锁: `Flush tables with read lock`
- 解锁: `unlock tables`，客户端断开时会自动释放锁
- 场景: 全库逻辑备份，即把整库每个表都 select 出来存成文本
- 加锁范围: 数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句都会被阻塞

做全库备份时，对于 Innodb，通过可重复度隔离级别我们就可以获取数据库的一致视图，但是对 于MyISAM 这些不支持事务的存储引擎，只能使用 `Flush tables with read lock` 让整个库处于只读状态。


既然要全库只读，为什么不使用 set global readonly=true 的方式呢？确实 readonly 方式也可以让全库进入只读状态，但我还是会建议你用 FTWRL 方式，主要有两个原因：
1. 在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，修改 global 变量的方式影响面更大
2. 在异常处理机制上有差异。如果执行 FTWRL 命令之后由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁，整个库回到可以正常更新的状态。而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，这样会导致整个库长时间处于不可写状态，风险较高。

## 2.表级锁
MySQL 表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。

表锁:
- 加锁: `lock tables t read/write`
	- tables 后面 s 可省略
	- `lock tables t read/writer` 会持有表 t 的 MDL 读/写锁
- 解锁: `unlock tables`，客户端断开时会自动释放锁
- 加锁范围: 除了会限制别的线程的读写外，也限定了本线程接下来的操作对象，如果在某个线程 A 中执行 lock tables t1 read, t2 write;线程 A 在执行 unlock tables 之前，也只能执行读 t1、读写 t2 的操作。连写 t1 都不允许，自然也不能访问其他表，线程A只能访问他锁定的表

元数据锁:
- 加锁: MDL 不需要显式使用，当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。MDL 的作用是，保证读写的正确性
- 解锁: 事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放

在做表结构变更的时候，如果操作不慎，就会导致锁住线上查询和更新。

![mdl](/images/mysql/MySQL45讲/mdl.jpg)

1. session A 先启动，这时候会对表 t 加一个 MDL 读锁
2. 由于 session B 需要的也是 MDL 读锁，因此可以正常执行
3. 因为 session A 的 MDL 读锁还没有释放，而 session C 需要 MDL 写锁，因此只能被阻塞
4. session D 等后续所有对表的增删改查操作都需要先申请 MDL 读锁，就都被锁住，等于这个表现在完全不可读写了
5. 如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新 session 再请求的话，这个库的线程很快就会爆满。

因为MDL锁只有在事务提交之后才会释放，因此对于存在长事务，或者操作非常频繁的表做DDL时要非常小心。好的做法是:
1. 在 MySQL 的 information_schema 库的 innodb_trx 表中，可以查到当前执行中的事务。要做 DDL 变更的表刚好有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务
2. 在 alter table 语句里面设定等待时间:

```bash
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 
```

**需要说明的是 5.7 版本修改了 MDL 的加锁策略，在>=5.7 的MySQL上是无法复现上面的场景的**。MDL 写锁在修改完表结构后就会退化为 MDL 读锁。

## 3.行锁
行锁就是针对数据表中行记录的锁。MySQL 的行锁是在引擎层由各个引擎自己实现的。MyISAM 引擎就不支持行锁。锁是加在索引上的，这是 InnoDB 的一个基础设定，在分析问题的时候一定要谨记。

### 3.1 两阶段锁
**在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议**。**如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放**。

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。如果把最可能造成冲突的锁语句放在最后面，这个锁的占用的时间就最少。这就最大程度地减少了事务之间的锁等待，提升了并发度。

### 3.2 死锁和死锁检测
当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。当出现死锁以后，有两种策略：
1. 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。
2. 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

但是这两种策略都各有利弊:
innodb_lock_wait_timeout: 默认值是 50s，对于在线服务来说，这个等待时间往往是无法接受的。我们又不可能直接把这个时间设置成一个很小的值，因为如果是简单的锁等待而不是死锁，过短的超时时间会造成很多误伤。正常情况下我们还是要采用第二种策略，即：主动死锁检测，而且 innodb_deadlock_detect 的默认值本身就是 on。

主动死锁检测: 在发生死锁的时候，是能够快速发现并进行处理的，但是它也是有额外负担的。每当一个事务被锁的时候，就要看看它所依赖的线程有没有被别人锁住，如此循环，最后判断是否出现了循环等待，也就是死锁。假如所有事务都要更新同一行，每个新来的被堵住的线程，都要判断会不会由于自己的加入导致了死锁，这是一个时间复杂度是 O(n) 的操作。假设有 1000 个并发线程要同时更新同一行，那么死锁检测操作就是 100 万这个量级的。虽然最终检测的结果是没有死锁，但是这期间要消耗大量的 CPU 资源。因此，你就会看到 CPU 利用率很高，但是每秒却执行不了几个事务。

#### 热点行更新导致的性能问题
热点行更新导致的性能问题的症结在于，死锁检测要耗费大量的 CPU 资源。解决办法有如下几种:
1. 如果你能确保这个业务一定不会出现死锁，可以临时把死锁检测关掉。但是这种操作本身带有一定的风险，因为业务设计的时候一般不会把死锁当做一个严重错误，毕竟出现死锁了，就回滚，然后通过业务重试一般就没问题了，这是业务无损的。而关掉死锁检测意味着可能会出现大量的超时，这是业务有损的。
2. 另一个思路是控制访问相同资源的并发事务量
	- 比如同一行同时最多只有 10 个线程在更新，那么死锁检测的成本很低。并发控制要做在数据库服务端。如果有中间件，可以考虑在中间件实现；如果能修改 MySQL 源码的人，也可以做在 MySQL 里面。基本思路就是，对于相同行的更新，在进入引擎之前排队。这样在 InnoDB 内部就不会有大量的死锁检测工作了。
	- 考虑通过将一行改成逻辑上的多行来减少锁冲突，但是需要根据业务逻辑做详细设计

## 4.MDL与主从同步问题
问题: 当备库用–single-transaction 做逻辑备份的时候，如果从主库的 binlog 传来一个 DDL 语句会怎么样？

假设这个 DDL 是针对表 t1 的，备份过程的语句如下:
```bash
Q1:SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ; # 确保 RR（可重复读）隔离级别
Q2:START TRANSACTION  WITH CONSISTENT SNAPSHOT；            # 得到一个一致性视图
/* other tables */     
Q3:SAVEPOINT sp;                  # 设置一个保存点
/* 时刻 1 */
Q4:show create table `t1`;        # 拿到表结构
/* 时刻 2 */
Q5:SELECT * FROM `t1`;            # 正式导数据 
/* 时刻 3 */
Q6:ROLLBACK TO SAVEPOINT sp;      # 释放 t1 的 MDL 锁 
/* 时刻 4 */
/* other tables */
```

DDL 从主库传过来的时间按照效果不同:
1. 如果在 Q4 语句执行之前到达，现象：没有影响，备份拿到的是 DDL 后的表结构。
2. 如果在“时刻 2”到达，则表结构被改过，Q5 执行的时候，报 Table definition has changed, please retry transaction，现象：mysqldump 终止；
3. 如果在“时刻 2”和“时刻 3”之间到达，mysqldump 占着 t1 的 MDL 读锁，binlog 被阻塞，现象：主从延迟，直到 Q6 执行完成。
4. 从“时刻 4”开始，mysqldump 释放了 MDL 读锁，现象：没有影响，备份拿到的是 DDL 前的表结构


## 5. 如何查看锁等待
如果一条语句长时间不返回，一般碰到这种情况的话，大概率是表 t 被锁住了。一般首先执行一下 show processlist 命令，看看当前语句处于什么状态。然后我们再针对每种状态，去分析它们产生的原因、如何复现，以及如何处理。下面我们将分成如下几种情况来分析如何查看和解决锁等待:
1. MDL 锁等待


### 5.1 MDL 锁等待

![waiter_mdl](/images/mysql/MySQL45讲/waiter_mdl.png)

show processlist 出现 Waiting for table metadata lock 时，表示的是，现在有一个线程正在表 t 上请求或者持有 MDL 写锁，导致 select 语句被堵住。


这类问题的处理方式，就是找到谁持有 MDL 写锁，然后把它 kill 掉。

但是，由于在 show processlist 的结果里面，session A 的 Command 列是“Sleep”，导致查找起来很不方便。不过有了 performance_schema 和 sys 系统库以后，就方便多了。（MySQL 启动时需要设置 performance_schema=on，相比于设置为 off 会有 10% 左右的性能损失)

![schema_table_lock_waits](/images/mysql/MySQL45讲/schema_table_lock_waits.png)

### 5.2 等待 flush

![waiter_flush](/images/mysql/MySQL45讲/waiter_flush.png)

这个状态表示的是，现在有一个线程正要对表 t 做 flush 操作。MySQL 里面对表做 flush 操作的用法，一般有以下两个，如果指定表 t 的话，代表的是只关闭表 t；如果没有指定具体的表名，则表示关闭 MySQL 里所有打开的表。

```bash
flush tables t with read lock;

flush tables with read lock;
```


但是正常这两个语句执行起来都很快，除非它们也被别的线程堵住了。所以，出现 Waiting for table flush 状态的可能情况是：有一个 flush tables 命令被别的语句堵住了，然后它又堵住了我们的 select 语句。这个例子很简单，通过 processlist 直接就可以看出导致阻塞的语句。


### 5.3 等行锁
![waiter_line](/images/mysql/MySQL45讲/waiter_line.png)

通过 `lock in share mode` 可以判断，sql 语句正在等待行锁。但问题是怎么查出是谁占着这个写锁。如果你用的是 MySQL 5.7 版本，可以通过 sys.innodb_lock_waits 表查到。

![nodb_lock_waits](/images/mysql/MySQL45讲/innodb_lock_waits.png)

blocking_pid 显示 4 号线程是造成堵塞的罪魁祸首。而干掉这个罪魁祸首的方式，就是 KILL QUERY 4 或 KILL 4。
1. KILL QUERY 4: 表示停止 4 号线程当前正在执行的语句，而这个方法其实是没有用的。因为占有行锁的是 update 语句，这个语句已经是之前执行完成了的，现在执行 KILL QUERY，无法让这个事务去掉 id=1 上的行锁
2. KILL 4: 表示直接断开这个连接。连接被断开的时候，会自动回滚这个连接里面正在执行的线程，也就释放了 id=1 上的行锁。



