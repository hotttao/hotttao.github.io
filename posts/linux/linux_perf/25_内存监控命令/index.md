# 4.5 内存监测工具


本节我们来介绍内存相关的监测工具。
<!-- more -->


## 1. 命令总览
下面的图片摘录自[极客时间专栏-Linux性能优化实战](https://time.geekbang.org/column/intro/140)，分别从下面 3 个方面总结了内存相关的性能检测工具:
1. 从内存的性能指标出发，根据指标找工具
2. 从工具出发，根据工具找指标
3. 根据工具指标之间的内在联系，掌握内存分析的套路

![memory_quota](/images/linux_pf/memory_quota.png)
![memory_command](/images/linux_pf/memory_command.png)
![memory_relation](/images/linux_pf/memory_relation.png)

有些工具是通用的分析工具，后面会在单独的章节中详细说明他们的使用。本节会介绍如下内存专用的分析工具的使用

|Linux|Solaris|作用|说明|
|:---|:---|:---|:---|
|vmstat|vmstat|虚拟和物理内存统计信息||
|slabtop|::kmastat|内核块分配统计信息||
|pmap|pmap|进程地址空间统计信息||
|pcstat|pcstat|查看文件在内存中的缓存大小以及缓存比例||
|cachetop|cachetop|实时查看间隔时间内每个进程的缓存命中情况|bcc工具包中|
|cachestat|cachestat|查看整个操作系统缓存的读写命中情况|bcc工具包中|
|memleak|memleak|内存泄漏跟踪|bcc工具包中|
|ps|ps|进程状态|通用命令，位于独立的一节中|
|top|prstat|监控进程内存使用|通用命令，位于独立的一节中|
|sar|sar|内存，swap使用统计信息|通用命令，位于独立的一节中|
|Systemtap|Dtrace|动态追踪|通用命令，位于独立的一节中|

除此之外，还包括以下内容:
1. 内存调优

## 2. 内存统计命令
### 2.1 vmstat
```bash
> vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st  
 2  0      0 1077816   2116 620312    0    0    65    43   34   53  0  0 99  0  0
# 说明:
# 除了 r 列外，第一行是系统启动以来的总结信息

> vmstat -a
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free  inact active   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 342188 472012 815924    0    0    68    84  160   72  9  1 90  0  0

> vmstat -s
  1882152 K total memory
   250252 K used memory
   815892 K active memory
   472012 K inactive memory
   341924 K free memory
      172 K buffer memory
  1289804 K swap cache
  2097148 K total swap
        0 K used swap
  2097148 K free swap
    99051 non-nice user cpu ticks
        0 nice user cpu ticks
    12143 system cpu ticks
   998690 idle cpu ticks
      360 IO-wait cpu ticks
        0 IRQ cpu ticks
      358 softirq cpu ticks
        0 stolen cpu ticks
   758858 pages paged in
   934798 pages paged out
        0 pages swapped in
        0 pages swapped out
  1777012 interrupts
   803881 CPU context switches
1586602557 boot time
    31172 forks
```

vmstat [t [n]]
- 作用: 虚拟内存统计命令
- 参数:
  - -t：采样间隔
  - -n：采样次数，可选，默认值是1
  - -S: -Sm 以MB 为单位显示结果
  - -a: 输出非活动和活动页缓存的明细
  - -s: 以列表显示内存统计信息
- 输出: 
  - r: 可运行线程数，所有等待加上正在运行的线程数，不包括处于不可中断睡眠状态的线程
  - cpu: 
    - 系统全局范围内的平均负载
    - 第一行统计的系统启动以来的平均负载 -- 新版本可能不会显示
    - 其余行统计的是时间间隔周期内的平均负载
  - memory:
    - swpd: 交换出的内存量
    - free: 空闲可用内存
    - buff: 用于缓冲缓存的内存
    - cache: 用于页缓存的内存
  - swap: 
    - si: 换入的内存
    - so: 换出的内存

### 2.2 slabtop
```bash
> sudo slabtop -sc
 Active / Total Objects (% used)    : 1079865 / 1085229 (99.5%)  # slab 管理的对象数量
 Active / Total Slabs (% used)      : 27647 / 27647 (100.0%)     # Slab 数量
 Active / Total Caches (% used)     : 69 / 97 (71.1%)            # 缓存的 slab 数量
 Active / Total Size (% used)       : 151423.96K / 153971.68K (98.3%) # slab 管理的内存大小
 Minimum / Average / Maximum Object : 0.01K / 0.14K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 50440  50440 100%    0.94K   6305        8     50440K xfs_inode
 81879  81087  99%    0.19K   3899       21     15596K dentry
 15392  15266  99%    1.00K   1924        8     15392K kmalloc-1024
139737 139737 100%    0.10K   3583       39     14332K buffer_head
 16419  16419 100%    0.58K   1263       13     10104K inode_cache
 17262  17262 100%    0.57K   1233       14      9864K radix_tree_node
```
slabtop:
- 作用: 内核内存信息统计，等同于 vmstat -m 
- 来源: slab 统计信息来自 /proc/slabinfo
- 参数
    - -d n：每n秒更新一次显示的信息，默认是每3秒；
    - -s S：指定排序标准进行排序；
    - -o：显示一次后退出
    - -V：显示版本


### 2.3 pmap
```bash
> sudo pmap -x 792|head
792:   /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000       4       4       0 r-x-- python2.7
0000000000600000       4       4       4 r---- python2.7
0000000000601000       4       4       4 rw--- python2.7
00000000017b4000    8800    8684    8684 rw---   [ anon ]
....

> sudo pmap -d 792 |head
792:   /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
Address           Kbytes Mode  Offset           Device    Mapping
0000000000400000       4 r-x-- 0000000000000000 0fd:00000 python2.7
0000000000600000       4 r---- 0000000000000000 0fd:00000 python2.7
0000000000601000       4 rw--- 0000000000001000 0fd:00000 python2.7
00000000017b4000    8800 rw--- 0000000000000000 000:00000   [ anon ]
00007fa0ac000000     132 rw--- 0000000000000000 000:00000   [ anon ]
.....
mapped: 359028K    writeable/private: 30856K    shared: 36K
```
pmap  PID
- 作用: 列出进程的内存映射，显示它们的大小，权限和映射对象
- 参数:
    - -x：显示扩展格式；
    - -d：显示设备格式；
    - -q：不显示头尾行；
    - -V：显示指定版本
- 输出:
    - pid -X
        - Address: 映射的起始地址
        - Kbytes: 虚拟内存大小
        - RSS: 主存大小
        - Dirty: 脏页大小
        - Mode: 映像权限: r=read, w=write, x=execute, s=shared, p=private  (copy on write) 
        - Mapping: 映像的文件,[anon]为已分配内存 [stack]为程序堆栈
    - pid -d
        - Offset: 文件偏移
        - Device: 设备名
        - mapped: 进程映射的虚拟地址空间大小，也就是该进程预先分配的虚拟内存大小，即ps出的vsz 
        - writeable/private: 进程所占用的私有地址空间大小，也就是该进程实际使用的内存大小   
        - shared: 表示进程和其他进程共享的内存大小

### 2.4 pcstat
pcstat 是一个基于 Go 语言开发的工具，所以安装它之前，首先需要[安装 Go 语言](https://rainbowmango.gitbook.io/go/chapter11/1.1-install)。由于不可描述的原因，go 包的安装可能会遇到问题，建议像下面这样设置一下代理后在安装:

```bash
# go 安装
cd /usr/local
wget https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.14.4.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' > /etc/profile.d/go.sh

# 设置代理 https://goproxy.io/zh/
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct

# 安装 pcstat
go get golang.org/x/sys/unix
go get github.com/tobert/pcstat/pcstat
```

`pcstat`
- 作用: 查看文件在内存中的缓存大小以及缓存比例
- 参数:

```bash
$ pcstat /bin/ls
+---------+----------------+------------+-----------+---------+
| Name    | Size (bytes)   | Pages      | Cached    | Percent |
|---------+----------------+------------+-----------+---------|
| /bin/ls | 133792         | 33         | 0         | 000.000 |
+---------+----------------+------------+-----------+---------+
```
指标含义:
1. Cached: /bin/ls 在缓存中的大小
2. Percent: 缓存的百分比

### 2.5 smem

### 2.6 其他命令
|命令|作用|
|:---|:---|
|dmesg|检查 OOM|
|valgrind|包含一个 memcheck 性能分析套件，可用于发现泄漏，<br>会有严重的系统开销
|swapon|添加和观察swap分区|
|iostat|如果 swap 是物理磁盘或块，可用此命令观测系统是否在换页|
|/proc/zoneinfo|内存区域 NUMA 节点的统计信息<br>NUMA 非均匀访存模型|
|/proc/buddyinfo|内核页面伙伴分配器统计信息<br>伙伴分配器:Linux 的页面分配器|

## 3. 内存调优
最重要的内存调优是保证应用程序保留在主存中，并且避免换页和交换经常发生。

### 3.1 可调参数
Documention/sysctl/vm.txt 的内核源码文档介绍了多种内存可调参数。常用示例如下

|参数|默认值|作用|
|:---|:---|:---|
|vm.dirty_background_ratio|10|触发pdflush后台回写的脏页百分比|
|vm.dirty_ratio|20|触发一个写入进程开始回写脏页比例|
|vm.dirty_expire_centisecs|3000|使用pdflush的脏存储器最小时间|
|vm.dirty_writeback_centisecs|5000|pdflush 活跃时间间隔，0为停用|
|vm.min_free_kbytes|dynamic|设置期望的空闲内存大小，dynamic为系统自动设置|
|vm.overcommit_memory|0|0: 利用探索算法允许合理的过度使用<br>1: 一致过度使用<br>3: 不允许过度使用<br>|
|vm.swappiness|60|相对于页面高速缓存回收更倾向于用交换释放内存的程度<br>高数值更倾向于交换应用程序而保留页缓存|


### 3.2 配置大页面
更大的页面能通过提高 TLB 缓存命令率，来提升内存 IO 性能。现在处理器支持多个页面大小。设置大页面(巨页面)参考文档hugetlbpage.txt。关于大页面的使用详见文档

### 3.3 分配器
应用程序的用户级分配器可以在编译阶段选择，也可以在执行时用 LD_PRELOAD 环境变量设置。

### 3.4 资源控制
主存限制和虚拟内存限制，可通过 ulimit 实现。Linux cgroup 内存子系统可提供多种附加控制:
1. memory.memsw.limit_in_bytes: 允许的最大内存和交换空间
2. memory.limit_in_bytes: 允许的最大用户内存，包括文件缓存
3. memory.swappiness: 类似 vm.swappiness，作用于 cgroups
4. memory.oom_control: 设置为 0，允许 OOM 应用于此 cgroup，1 不允许 
