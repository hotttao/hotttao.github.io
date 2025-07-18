# 13 MYSQL 索引选择


MySQL 如何选择索引

<!-- more -->

## 1. 优化器逻辑
选择索引是优化器的工作。而优化器选择索引的目的，是找到一个最优的执行方案，并用最小的代价去执行语句。优化器会考虑扫描行数、是否使用临时表、是否排序等因素进行综合判断。对于普通索引来说，优化器需要把回表的代价算进去。

## 2. 扫描行数
MySQL 在真正开始执行语句之前，并不能精确地知道满足这个条件的记录有多少条，而只能根据统计信息来估算记录数。这个统计信息就是索引的“区分度”。
1. 一个索引上不同的值越多，这个索引的区分度就越好。
2. 一个索引上不同的值的个数，我们称之为“基数”（cardinality）。这个基数越大，索引的区分度越好。

### 2.1 索引区分度
`show index from t` 可以查看索引的基数。

```bash
MariaDB [tsong]> show index from course \G
*************************** 1. row ***************************
        Table: course
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 3                  # 基数
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
```

那 mysql 是如何计算索引的基数呢？ mysql 用的采样统计的方法:
1. InnoDB 默认会选择 N 个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数
2. 当变更的数据行数超过 1/M 的时候，会自动触发重新做一次索引统计
3. MySQL 中，有两种存储索引统计的方式，通过设置参数 innodb_stats_persistent 的值来选择：
	- =ON: 表示统计信息会持久化存储。这时，默认的 N 是 20，M 是 10
	- =OFF: 表示统计信息只存储在内存中。这时，默认的 N 是 8，M 是 16

因为是采样统计，所以这个基数是很容易不准确的，如果索引统计不准确，可以使用 `analyze table t` 重新统计索引信息。在实践中，如果你发现 explain 的结果预估的 rows 值跟实际情况差距比较大，可以采用这个方法来处理。

索引统计只是一个输入，对于一个具体的语句来说，优化器还要判断，执行这个语句本身要扫描多少行。

### 2.2 索引选择异常和处理
如果只是索引统计不准确，通过 analyze 命令可以解决很多问题，但优化器可不止是看扫描行数。即便索引统计准确，MySQL 照样有可能选错索引。索引选择异常有以下几种处理方法:
1. 采用 force index 强行选择一个索引，使用 force index 最主要的问题是变更的及时性
2. 修改语句，引导 MySQL 使用我们期望的索引
3. 我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。

## 3. MySQL 选错索引的示例
我们创建如下表，并往表内插入10万行数据。
```bash

CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB；


delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into t values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

我们做如下操作:

![index_choose_error](/images/mysql/MySQL45讲/index_choose_error.png)

这时候，session B 的查询语句 select * from t where a between 10000 and 20000 就不会再选择索引 a 了。作为对比我们使用 force index(a) 来让优化器强制使用索引 a。下面的三条 SQL 语句，就是这个实验过程。

```bash
set long_query_time=0;
select * from t where a between 10000 and 20000; /*Q1*/
select * from t force index(a) where a between 10000 and 20000;/*Q2*/
```

我们可以通过通过慢查询日志（slow log）查看这两个SQL的执行过程:

![index_slow_qeury](/images/mysql/MySQL45讲/index_slow_qeury.png)

前面我们说过优化器在选择索引时会考虑扫描行数、是否使用临时表、是否排序等因素进行综合判断。而扫描行数与索引区分度有关。这个简单的查询语句并没有涉及到临时表和排序，所以是扫描行数出了问题。使用 explain 可以查看优化器预估的，这两个语句的扫描行数:

![10_index_compare](/images/mysql/MySQL45讲/10_index_compare.png)

这里有两个问题:
1. 使用 force index(a) 的扫描行数为什么不准，达到了 37000？
2. 优化器为什么放着扫描 37000 行的执行计划不用，却选择了扫描行数是 100000 的执行计划呢？

选择100000 的执行计划是因为使用普通索引需要把回表的代价算进去。优化器认为直接扫描主键索引更快。

而扫描行数不准与我们的操作序列有关。session A 开启了事务并没有提交，所以之前插入的 10 万行数据是不能删除的。这样，之前的数据每一行数据都有两个版本，旧版本是 delete 之前的数据，新版本是标记为 deleted 的数据。这样，索引 a 上的数据其实就有两份。这个例子对应的是我们平常不断地删除历史数据和新增数据的场景。

你可能觉得主键上的数据也不能删，那没有使用 force index 的语句，使用 explain 命令看到的扫描行数为什么还是 100000 左右？（潜台词，如果这个也翻倍，也许优化器还会认为选字段 a 作为索引更合适）是的，不过这个是主键，主键是直接按照表的行数来估计的。而表的行数，优化器直接用的是 show table status 的值。

![primary_key_num](/images/mysql/MySQL45讲/primary_key_num.png)

统计信息不对，我们可以使用 analyze table t 命令，重新统计索引信息。当然并不是所有的索引选择错误都是由统计信息不对导致的，这时候我们就要通过上面说到的其他方法来解决。

