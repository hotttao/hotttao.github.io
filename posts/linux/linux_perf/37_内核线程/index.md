# 4.17 总结-内核线程


本节我们来看看，操作系统内都有哪些内核线程。
<!-- more -->

## 1. 内核线程
Linux 中，用户态进程的“祖先”，都是 PID 号为 1 的 init 进程(或者 systemd)。那么，内核态线程又是谁来管理的呢？

实际上，Linux 在启动过程中，有三个特殊的进程，也就是 PID 号最小的三个进程。
1. 0 号进程为 idle 进程，这也是系统创建的第一个进程，它在初始化 1 号和 2 号进程后，演变为空闲任务。当 CPU 上没有其他任务执行时，就会运行它。
2. 1 号进程为 init 进程，通常是 systemd 进程，在用户态运行，用来管理其他用户态进程。
3. 2 号进程为 kthreadd 进程，在内核态运行，用来管理内核线程。

所以可以像下面这样查看所有内核线程:
```bash
# 1. 通过 2 号进程查看所有内核线程
$ ps -f --ppid 2 -p 2
UID         PID   PPID  C STIME TTY          TIME CMD
root          2      0  0 12:02 ?        00:00:01 [kthreadd]
root          9      2  0 12:02 ?        00:00:21 [ksoftirqd/0]
root         10      2  0 12:02 ?        00:11:47 [rcu_sched]
root         11      2  0 12:02 ?        00:00:18 [migration/0]
...
root      11094      2  0 14:20 ?        00:00:00 [kworker/1:0-eve]
root      11647      2  0 14:27 ?        00:00:00 [kworker/0:2-cgr]

# 2. 内核线程的名称（CMD）都在中括号里
$ ps -ef | grep "\[.*\]"
root         2     0  0 08:14 ?        00:00:00 [kthreadd]
root         3     2  0 08:14 ?        00:00:00 [rcu_gp]
root         4     2  0 08:14 ?        00:00:00 [rcu_par_gp]
...
```

### 1.1 常见的内核线程
性能分析中经常会碰到如下几个内核线程:
1. ksoftirqd: 处理软中断的内核线程，每个 CPU 上都有一个
2. kswapd0：用于内存回收
3. kworker：用于执行内核工作队列，分为
    - 绑定 CPU （名称格式为 kworker/CPU86330）和
    - 未绑定 CPU（名称格式为 kworker/uPOOL86330）两类
4. migration：在负载均衡过程中，把进程迁移到 CPU 上，每个 CPU 一个
5. jbd2/sda1-8：
    - jbd 是 Journaling Block Device 的缩写
    - 用来为文件系统提供日志功能，以保证数据的完整性；
    - 名称中的 sda1-8，表示磁盘分区名称和设备
    - 每个使用了 ext4 文件系统的磁盘分区，都会有一个 jbd2 内核线程
6. pdflush：用于将内存中的脏页（被修改过，但还未写入磁盘的文件页）写入磁盘（已经在 3.10 中合并入了 kworker 中）


## 2. 内核线程性能剖析
对于普通进程，我们要观察其行为有很多方法，比如 strace、pstack、lsof 等等。但这些工具并不适合内核线程，比如，如果你用 pstack ，或者通过 /proc/pid/stack 查看 ksoftirqd/0（进程号为 9）的调用栈时，分别可以得到以下输出：

```bash
# pstack 报出的是不允许挂载进程的错误
$ pstack 9
Could not attach to target 9: Operation not permitted.
detach: No such process

# /proc/9/stack 方式虽然有输出，但输出中并没有详细的调用栈情况。
$ cat /proc/9/stack
[<0>] smpboot_thread_fn+0x166/0x170
[<0>] kthread+0x121/0x140
[<0>] ret_from_fork+0x35/0x40
[<0>] 0xffffffffffffffff
```

内核的追踪，我们需要借助动态追踪技术，比如 perf，Systemtap, eBPF。借助于这些工具我们可以生成火焰图，帮助我们分析各种问题。各种工具的使用，我们在这个系列文章的开始都作了详细介绍。希望大家多多练习，熟练掌握。下面我举几个例子，让大家看看，这些工具是如何达到异曲同工之妙的。

## 3. 跟踪网络丢包


## 4. 火焰图追踪 ksoftirqd 调用栈
