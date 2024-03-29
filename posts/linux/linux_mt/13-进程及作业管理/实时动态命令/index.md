# 12.3 进程管理的实时动态命令


进程管理实时动态命令

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

下面介绍的 top, htop，glances 都是能实时查看系统状态的动态命令，有众多快捷键可以控制屏幕显示的内容。htop 是 top 命令的升级版，glance 则是另一款 htop 它们实现的功能类似。

## 1. top
`uptime`
- 作用:
    - 显示系统时间、运行时长及平均负载；
    - 过去1分钟、5分钟和15分钟的平均负载；
    - 等待运行的进程队列的长度；
- 注释: 显示内容即 top 命令的首部信息

`top options`
- 作用: display Linux processe
- 选项:
    - `-d #`: 指定刷新时间间隔，默认为3秒；
    - `-b`: 以批次方式显示；
    - `-n #`: 显示多少批次，与 -b 一起使用；
- 快捷键:
    - 排序:
        - `P`: 以占据CPU百分比排序；
        - `M`: 以占据内存百分比排序；
        - `T`: 累积占用CPU时间排序；
    - 首部信息:
        - uptime信息: `l` 命令
        - tasks及cpu信息: `t` 命令
        - CPU分别显示使用数字 `1`
        - 内存信息: m命令
    - 退出命令: `q`
    - 修改刷新时间间隔: `s`
    - 终止指定的进程: `k`
    - 帮助: `h`
    - 显示COMMAND 详细信息: `c`

```
#快捷键: l# top - 18:10:50 up  9:21,  6 users,  load average: 0.73, 0.55, 0.40
#快捷键: t# Tasks: 333 total,   1 running, 332 sleeping,   0 stopped,   0 zombie
#快捷键: 1# %Cpu(s):  5.6 us,  2.3 sy,  0.0 ni, 90.8 id,  0.1 wa,  0.8 hi,  0.4 si,  0.0 st
#快捷键: m# KiB Mem :  8115092 total,   501932 free,  5477676 used,  2135484 buff/cache
           KiB Swap:  2097148 total,  2093124 free,     4024 used.  1486672 avail Mem

           PID USER    PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND          
           4812 tao    20   0 2876852 431600 139652 S   9.5  5.3   49:00.29 atom          
           2150 root   20   0  447256 122816  71820 S   4.6  1.5   13:42.31 X
           4384 tao    20   0 4824324 1.066g  39644 S   3.6 13.8   49:54.58 java
```

`top - 18:10:50 up  9:21,  6 users,  load average: 0.73, 0.55, 0.40`
- 作用: Top 任务队列信息(系统运行状态及平均负载)，与uptime命令结果相同
- 字段:
  - 系统当前时间
  - 系统运行时间，未重启的时间
  - 当前登录用户数
  - 系统负载，即任务队列的平均长度，3个数值分别统计最近1，5，15分钟的系统平均负载
    - 单核CPU情况下，0.00 表示没有任何负荷，1.00表示刚好满负荷，超过1侧表示超负荷，理想值是0.7；
    - 多核CPU负载：CPU核数 * 理想值0.7 = 理想负荷，例如：4核CPU负载不超过2.8何表示没有出现高负载

`Tasks: 333 total,   1 running, 332 sleeping,   0 stopped,   0 zombie`
- 作用: Tasks 进程相关信息
- 字段:
  - 进程总数，例如：Tasks: 231 total, 表示总共运行231个进程
  - 正在运行的进程数，例如：1 running,
  - 睡眠的进程数，例如：230 sleeping,
  - 停止的进程数，例如：0 stopped,
  - 僵尸进程数，例如：0 zombie

`%Cpu(s):  5.6 us,  2.3 sy,  0.0 ni, 90.8 id,  0.1 wa,  0.8 hi,  0.4 si,  0.0 st`
- 作用: CPU 相关信息
- 字段:
  - `us`: 用户空间占用CPU百分比，例如：Cpu(s): 12.7%us,
  - `sy`: 内核空间占用CPU百分比，例如：8.4%sy,
  - `ni`: 用户进程空间内改变过优先级的进程占用CPU百分比，例如：0.0%ni,
  - `id`: 空闲CPU百分比，例如：77.1%id,
  - `wa`: 等待输入输出的CPU时间百分比，例如：0.0%wa,
  - `hi`: CPU服务于硬件中断所耗费的时间总额，例如：0.0%hi,
  - `si`: CPU服务软中断所耗费的时间总额，例如：1.8%si,
  - `st`: Steal time 虚拟机被hypervisor偷去的CPU时间（如果当前处于一个hypervisor下的vm，实际上hypervisor也是要消耗一部分CPU处理时间的）

`KiB Mem :  8115092 total,   501932 free,  5477676 used,  2135484 buff/cache`  
`KiB Swap:  2097148 total,  2093124 free,     4024 used.  1486672 avail Mem`
- 作用: 内存 相关信息
- 字段:
  - 物理内存总量，例如：Mem: 12196436k total,
  - 使用的物理内存总量，例如：12056552k used,
  - 空闲内存总量，例如：Mem: 139884k free,
  - 用作内核缓存的内存量，例如：64564k buffers

字段值含义
- `PID = (Process Id)`: 进程Id；
- `USER = (User Name)`: 进程所有者的用户名；
- `PR = (Priority)`: 优先级
- `NI = (Nice value)`: nice值。负值表示高优先级，正值表示低优先级
- `VIRT = (Virtual Image (kb))`: 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
- `RES = (Resident size (kb))`: 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
- `SHR = (Shared Mem size (kb))`: 共享内存大小，单位kb
- `S = (Process Status)`: 进程状态。D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程
- `%CPU = (CPU usage)`: 上次更新到现在的CPU时间占用百分比
- `%MEM = (Memory usage (RES))`: 进程使用的物理内存百分比
- `TIME+ = (CPU Time, hundredths)`: 进程使用的CPU时间总计，单位1/100秒
- `PPID = (Parent Process Pid)`: 父进程Id
- `RUSER = (Real user name)`:
- `UID = (User Id)`: 进程所有者的用户id
- `GROUP = (Group Name)`: 进程所有者的组名
- `TTY = (Controlling Tty)`: 启动进程的终端名。不是从终端启动的进程则显示为 ?
- `P = (Last used cpu (SMP))`: 最后使用的CPU，仅在多CPU环境下有意义
- `SWAP = (Swapped size (kb))`: 进程使用的虚拟内存中，被换出的大小，单位kb
- `TIME = (CPU Time)`: 进程使用的CPU时间总计，单位秒
- `CODE = (Code size (kb))`: 可执行代码占用的物理内存大小，单位kb
- `DATA = (Data+Stack size (kb))`: 可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
- `nFLT = (Page Fault count)`: 页面错误次数
- `nDRT = (Dirty Pages count)`: 最后一次写入到现在，被修改过的页面数
- `WCHAN = (Sleeping in Function)`: 若该进程在睡眠，则显示睡眠中的系统函数名
- `Flags = (Task Flags <sched.h>)`: 任务标志，参考 sched.h
- `COMMAND = (Command name/line)`: 命令名/命令行

## 2. htop
`htop options`
- 选项:
    - `-d #`: 指定延迟时间间隔；
    - `-u UserName`: 仅显示指定用户的进程；
    - `-s COLUME`: 以指定字段进行排序；
- 子命令:
    - `l`: 显示选定的进程打开的文件列表；
    - `s`: 跟踪选定的进程的系统调用；
    - `t`: 以层级关系显示各进程状态；
    - `a`: 将选定的进程绑定至某指定的CPU核心；


## 3. glances命令：
`glances options`
- 作用: 动态的系统状态监控工具，使用类似 top
- 常用选项：
    - `-b`：以Byte为单位显示网上数据速率；
    - `-d`：关闭磁盘I/O模块；
    - `-m`：关闭mount模块；
    - `-n`：关闭network模块；
    - `-t #`：刷新时间间隔；
    - `-1`：每个cpu的相关数据单独显示；
    - `-o {HTML|CSV}`：输出格式；
    - `-f /PATH/TO/SOMEDIR`：设定输出文件的位置；
- C/S模式下运行glances命令：
    - 服务模式：`glances  -s  -B  IPADDR`
        - IPADDR：本机的某地址，用于监听；
    - 客户端模式： `glances  -c  IPADDR`
        - IPADDR：是远程服务器的地址；
    - 附注: C/S 模式下无论是密码还是内容都是明文传输的，容易被截获，glances 不适合 C/S 模式使用

