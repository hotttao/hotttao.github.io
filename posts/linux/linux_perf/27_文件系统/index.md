# 4.7 文件系统


本节我们来介绍文件系统相关的操作系统原理。文件系统性能比磁盘性能更加重要。文件系统通过缓存，缓冲以及异步I/O等手段来缓和了磁盘延时对应用程序的影响。
<!-- more -->

## 1. 文件系统相关的操作原理
我们从下面几个方面入手来讲解文件系统相关的操作系统原理:
1. 文件系统I/O栈
2. 文件系统缓存
3. I/O的多种方式

最后我们会说一说文件系统检测的相关指标。

## 2. 文件系统 I/O 栈
下面是文件系统I/O栈的一般模型，具体的模块和层次依赖于使用的操作系统。

![io_stack](/images/linux_pf/io_stack.png)
- Volume Manager: 卷管理器
- Block Device Interface: 块设备接口
- Host Bus Adaptor Driver: 主机总线适配器驱动
- Disk Devices: 磁盘设备 

### 2.1 VFS
虚拟文件系统 VFS 是一个文件系统类型作抽象的内核界面。VFS 接口让内核添加新的文件系统时更加简单。

![vfs](/images/linux_pf/vfs.png)

VFS 接口可以作为测量文件系统性能的通用平台，能够利用操作系统提供的统计信息，静态及动态追踪技术。

## 3. 文件系统缓存
UNIX 原本只有缓冲区高速缓存，如今 Linux 和 Solaris 都有多种缓存。

### 3.1 Solaris 的文件系统缓存
下图是基于 Solaris 系统的文件系统缓存的概览。

![files_system_cache](/images/linux_pf/files_system_cache.png)

其中有三种缓存是文件系统里通用的，其他的都是每个文件系统特有的，这三种缓存包括:
1. Old Buffer Cache: 旧式的缓冲区高速缓存
2. page Cache: 页缓存
3. DNLC: 目录名查找缓存

#### 旧式的缓冲区高速缓存(buffer cache)
最初 UNIX 在块设备接口使用缓冲区高速缓存来缓存**磁盘设备块**。页缓存的加入带来了优化问题，比如:
1. 如何平衡二者之间的负载
2. 双重缓存和同步开销

这些问题后来基本被 SunOS 中的统一缓冲区高速缓存解决了，方法是使用**页缓存来存储缓冲区高速缓存**

在 Solaris 旧式的缓冲区高速缓存依然存在，不过仅用于 UFS inode 和文件系统的元数据，这些数据通过它们的块号寻址，与文件无关。

#### 页缓存
页缓存就是我们在内存一章所说的文件系统页缓存。它缓存了虚拟内存页面映射过的文件系统页面。

#### DNLC 
DNLC 目录名查找缓存记录了目录项到 vnode 的映射关系。用于加速文件的查找。

### 3.2 Linux 的文件系统缓存
与 Solaris 类似，Linux 也经历旧式的缓冲区高速缓存到统一缓冲区高速缓存的过程。方法也是类似的，即缓冲区高速缓存被存在了页缓存中。

缓冲区高速缓存的功能仍在，用于缓存文件系统的元数据，提升了块设备I/O的性能。

![linux_file_cache](/images/linux_pf/linux_file_cache.png)

#### 写回缓存
写回缓存的原理是当数据写入主存(页缓存)后，就认为写入已经结束并返回，之后再异步的把数据刷入磁盘。文件系统写入"脏页"的过程称为刷新，就是我们经常说的刷脏页。

刷新的机制牺牲了可靠性，因为基于DRAM的主存是不可靠的，主机掉电写入的数据就会丢失。应用程序可能认为数据写入完成，但是实际上未被写入，甚至不完整写入。如果文件系统的元数据遭到破坏，可能无法加载，对业务造成严重的影响。

为了平衡系统对于速度和可靠性的需求，文件系统默认采用**写回缓存**，但同时提供了同步写的选项绕过这个机制。

#### 页缓存管理
文件系统使用的内存脏页由内核线程写回到磁盘，现在这个写回线程为 flusher thread，线程名为 flush，每个设备分配一个线程。这样能平衡每个设备的负载，提高吞吐量。

写脏页的时机包括:
1. 过了一段时间(30s)
2. 调用了 sync()，fsync(), msync() 等系统调用
3. 过多的脏页(dirty_ratio)
4. 页缓存没有可用的页面

如果系统内存不足，另一个内核线程，页面换出守护进程kswapd() 会定位并安排把脏页面写入到磁盘上，腾出可重用的内存页面。

#### 目录项缓存和 inode 缓存
目录项缓存记录了从目录项到 VFS inode 的映射关系。inode 缓存缓存的对象是 VFS inode，每一个都描述了文件系统一个对象的属性。

内核使用 Slab 机制，管理目录项和索引节点的缓存。通过 `/proc/slabinfo` 可以查看到各种 inode与目录项缓存的大小:

```bash

$ cat /proc/slabinfo | grep -E '^#|dentry|inode' 
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail> 
xfs_inode              0      0    960   17    4 : tunables    0    0    0 : slabdata      0      0      0 
... 
ext4_inode_cache   32104  34590   1088   15    4 : tunables    0    0    0 : slabdata   2306   2306      0hugetlbfs_inode_cache     13     13    624   13    2 : tunables    0    0    0 : slabdata      1      1      0 
sock_inode_cache    1190   1242    704   23    4 : tunables    0    0    0 : slabdata     54     54      0 
shmem_inode_cache   1622   2139    712   23    4 : tunables    0    0    0 : slabdata     93     93      0 
proc_inode_cache    3560   4080    680   12    2 : tunables    0    0    0 : slabdata    340    340      0 
inode_cache        25172  25818    608   13    2 : tunables    0    0    0 : slabdata   1986   1986      0 
dentry             76050 121296    192   21    1 : tunables    0    0    0 : slabdata   5776   5776      0 
```
指标含义:
1. inode_cache: VFS 索引节点缓存
2. dentry 行表示目录项缓存
3. 其余的则是各种文件系统的索引节点缓存。

### 3.3 buffer 和 cache
目前为止我们在两个地方提到了 buffer，cache:
1. 第一地方是页缓存，我们说页缓存统一了缓冲区高速缓存(buffer cache)。
2. 第二地方是内存一节我们提到 `/proc/meminfo` 文件记录了各种内存指标的统计信息，其中包括 Buffers 和 Cached

```bash
cat /proc/meminfo
MemTotal:        2895444 kB
MemFree:         2498868 kB
MemAvailable:    2535384 kB
Buffers:            3108 kB
Cached:           165872 kB
.....
```

那么问题是buffer, cache, 页缓存与Buffers, Cached 之间到底有什么关系。

`man proc` 可以看到 `/proc/meminfo` 文件内各个字段的准确含义:
1. Buffers: 
    - 是对原始磁盘块的临时存储，也就是用来缓存磁盘的数据，通常不会特别大（20MB 左右）。
    - 通过 Buffer 内核就可以把分散的写集中起来，统一优化磁盘的写入，比如可以把多次小的写合并成单次大的写等等
2. Cached: 是从磁盘读取文件的页缓存，也就是用来缓存从文件读取的数据

以前看到的文章说 Buffer 是对将要写入磁盘数据的缓存，Cache 是对从文件读取数据的缓存。但事实上:
1. Buffers 既可以用作“将要写入磁盘数据的缓存”，也可以用作“从磁盘读取数据的缓存”。
2. Cached 既可以用作“从文件读取数据的页缓存”，也可以用作“写文件的页缓存”

简单来说， Buffers 是对磁盘数据的缓存，而 Cached 是文件数据的缓存，它们既会用在读请求中，也会用在写请求中。

页缓存统一了缓冲区高速缓存(buffer cache) 但是并没有改变它的作用: 
1. 缓冲区高速缓存(buffer cache)用来缓存**磁盘设备块的写和读**，
2. 被用作缓冲区高速缓存(buffer cache) 的页缓存保存的是磁盘设备块
3. 对磁盘的直接读写，数据以磁盘设备块，保存在用作缓冲区高速缓存(buffer cache) 的页缓存中
4. 对文件系统的读写，数据以文件内容，保存在页缓存中
5. 文件内容和磁盘设备块不会同时保存，因此才避免了双重缓存和同步开销
6. Buffers 统计的是用作缓冲区高速缓存(buffer cache) 的页缓存大小
7. Cached 统计的是文件系统的页缓存大小

最后需要说明的是，free，top，vmstat，显示的 buffer/cache 依计算规则不同而有所不同，但是数据都来自`/proc/meminfo` 文件。

## 4. I/O 的多种方式
### 4.1 顺序与随机I/O
按照I/O的文件偏移量，I/O分为:
1. 顺序 I/O: 顺序I/O里每个 I/O 都开始于上一个 I/O 结束的地址
2. 随机 I/O: 随机 I/O则找不出I/O之间的关系

由于存储设备的性能特性，顺序I/O的速度要远远高于随机I/O。文件系统可以测量逻辑I/O的访问模式，从中识别出顺序I/O，然后通过预取或者预读来提高性能。

#### 预取
预取指当文件系统检测出当前为顺序读负载时，在应用程序请求前向磁盘发出读指令，以填充文件系统缓存。如果应用程序真的发出顺序读请求，就会命中缓存。

预期一旦命中读性能将会有显著提升，但是如果预测不准，文件系统会发起应用程序不需要的I/O，不仅污染了缓存，也消耗了磁盘和I/O传输的资源。

预取一般被认为是预读。Linux 的 readahead(2) 系统调用允许应用程序显示的预热文件系统缓存，此时二者就不同了。

### 4.2 同步写和非阻塞I/O
在前面的文件系统缓存，我们提到了文件系统的写回缓存和 buffer 机制，为了控制文件写入过程，操作系统提供了同步写。同步用于指绕过写回缓存机制，写操作必须等待至所有的数据以及必要的文件系统元数据完整的写入到存储设备中。写操作支持如下标识:
1. O_DSYNC: 表示，写操作必须要等文件数据写入磁盘后，才能返回；
2. O_SYNC: 则是在 O_DSYNC 基础上，要求文件元数据也要写入磁盘后，才能返回。

### 4.3 裸I/O和直接I/O
![raw_io](/images/linux_pf/raw_io.jpg)
1. 裸I/O: 
    - 如上图所示，绕过整个文件系统，直接发送磁盘地址
    - 数据库会使用裸I/O，因为它们能比文件系统更好地缓存自己的数据
2. 直接I/O: 
    - 绕过缓存使用文件系统
    - 可用于备份文件系统，放置污染文件系统缓存
裸I/O和直接I/O还可以用于那些在进程堆里自建缓存的应用程序，避免双重缓存的问题。kafka 应该就是舍弃了堆缓存，直接使用了操作系统的页缓存。

### 4.4 内存映射文件
内存映射文件可以把文件映射到进程地址空间，并直接存取内存地址的方法来提高文件系统I/O性能。这样可以避免调用 read() 和 write() 存取文件数据时产生的系统调用和上下文切换开下。

如果内核支持直接复制文件数据缓冲到进程进程地址空间，那么还能防止**数据被复制两次**。(It can also avoid
double copying of data, if the kernel supports direct copying of the file data buffer
to the process address space)
(数据为什么会复制两次: 意思是如果 mmap 映射的文件正在被另一进程写，它们是无法共享相同的页缓存的？)


内存映射文件通过系统调用 mmap() 创建，通过 munmap() 销毁，映射可以通过 madvise() 调整。

如果系统问题是由于磁盘设备高I/O延时所至，用 mmap() 消除小小的系统调用是无济于事的。

在多处理器上使用内存映射文件的缺点在于同步每个 CPU MMU 的开销。尤其是跨 CPU 的映射删除调用(TLB 击落)。延时 TLB 更新可能把影响最小化，这取决于内核和映射项。(A disadvantage of using mappings on multiprocessor systems can be the overhead to keep each CPU MMU in sync, specifically the CPU cross calls to remove
mappings (TLB shootdowns). Depending on the kernel and mapping, these may be
minimized by delaying TLB updates (lazy shootdowns) [Vahalia 96])

### 4.5 逻辑I/O 与物理I/O
- 逻辑I/O 指向文件系统发起的I/O
- 物理I/O: 磁盘I/O
与应用程序I/O相比，磁盘I/O有时显得无关、间接、放大或者缩小。

## 6. 文件系统监测指标
与 CPU 相关的专业术语或者指标包括:
1. 文件系统延时
2. 

### 6.1 文件系统延时
文件系统延时指的是一个文件系统逻辑从开始到结束的时间，它包括消耗在文件系统，内核磁盘I/O子系统以及等待磁盘设备(物理I/O)的时间。

文件系统延时是否会影响应用程序，取决于应用程序的 I/O 方式，同步I/O会有直接影响，非阻塞I/O或异步I/O则不会。文件系统一直以来未开放查看文件系统延时的接口。相反提供了磁盘设备级别的指标信息。但是多数情况下这些指标跟应用程序并无直接关系。原因同样是实际发生的物理I/O并不与应用程序的执行同步。




