# 12.4 系统状态查看


系统状态查看

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

系统状态查看命令，可以查看系统包括磁盘，CPU，内存，缓存，进程，网络等等几乎所有的状态信息。本节我们主要介绍 `vmstat`,`pmap`, `dstat` 三个命令的使用。

## 1. vmstat命令：
`vmstat  [options]  [delay [count]]`
- 作用: 查看虚拟内存使用状况
- 选项：
    - `-s`：显示内存统计数据；类似于 cat /proc/meminfo

```
tao@hp:~$ vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 484884   1292 3710880    0    0   179    83  455  865 14  4 82  1  0

```
输出:
1. `procs`：
    - `r`：等待运行的进程的个数；CPU上等待运行的任务的队列长度；
    - `b`：处于不可中断睡眠态的进程个数；被阻塞的任务队列的长度；
2. `memory`：
    - `swpd`：交换内存使用总量；
    - `free`：空闲的物理内存总量；
    - `buffer`：用于buffer的内存总量；
    - `cache`：用于cache的内存总量；
- `swap`
    - `si`：数据进入swap中的数据速率（kb/s）
    - `so`：数据离开swap的速率(kb/s)
- `io`
    - `bi`：从块设备读入数据到系统的速度（kb/s）
    - `bo`：保存数据至块设备的速率（kb/s）
- `system`
    - `in`：interrupts，中断速率，即每秒发生的中断次数；
    - `cs`：context switch, 上下文切换的速率，即每秒发生的进程切换次数；
- `cpu`
    - `us`：user space，用户空间占用 cpu 比例
    - `sy`：system，内核空间占用 cpu 比例
    - `id`：idle，cpu 空闲比例
    - `wa`：wait，等待 I/O 完成，消耗的 cpu 时间比例
    - `st`: stolen，被虚拟化技术偷走的 cpu 时间比例


## 2. pmap命令：
`pmap [options] pid [...]`
- 作用: 显示进程虚拟内存到物理内存的映射关系表
- `-x`：显示详细格式的信息；
- 另一种查看方式：`cat  /proc/PID/maps`


## 3. dstat命令：
`dstat [-afv] [options..] [delay [count]]`            
- 作用: 统计系统资源的使用情况
- 常用选项：
    - `-c， --cpu`：显示cpu相关信息；
    - `-C 1,3,...,total`: 显示指定的cpu相关信息；
    - `-d, --disk`：显示磁盘的相关信息
    - `-D sda,sdb,...,tobal`: 显示指定磁盘的相关信息
    - `-g`：显示page相关的速率数据；
    - `-m`：Memory的相关统计数据
    - `-n`：显示network 相关统计数据；
    - `-p`：显示process的相关统计数据；
    - `-r`：显示io请求的相关的统计数据；
    - `-s`：显示swapped的相关统计数据；
    - `--tcp`
    - `--udp`
    - `--raw`
    - `--socket`
    - `--ipc`
    - `--top-cpu`：显示最占用CPU的进程；
    - `--top-io`：最占用io的进程；
    - `--top-mem`：最占用内存的进程；
    - `--top-lantency`：延迟最大的进程；

```
tao@hp:~$ dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
 14   3  82   1   1   0|1355k  633k|   0     0 |   0     0 |1788  3408
 26   1  73   0   1   0|   0  4096B| 471B  224B|   0     0 |1916  1627
 26   0  73   1   0   0|   0   188k|  92B    0 |   0     0 |1950  1744
```

