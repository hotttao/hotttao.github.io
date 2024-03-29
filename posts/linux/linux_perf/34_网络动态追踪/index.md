# 4.15 网络动态追踪


本节我们来介绍网络的动态追踪技术
<!-- more -->

## 1. Systemtab 磁盘网络
Dtrace/Systemtab 可以在内核和应用程序内部检查网络事件，包括
1. 套接字连接
2. 套接字I/O
3. TCP事件
4. 数据包重传
5. 积压队列丢包
6. TCP重传

这些功能能够支持工作负载特征归纳和延时分析

### 1.1 Dtrace
下面列出了用来跟踪网络的 Dtrace provider
|层次|稳定 provider|不稳定 provider|
|:---|:---|:---|
|应用程序|取决于应用|pid|
|系统库||pid|
|系统调用||syscall|
|套接字||fbt|
|TCP|tcp,mib|fbt|
|UDP|udp,mib|fbt|
|IP|ip,mib|fbt|
|链路层||fbt|
|设备驱动||fbt|

### 1.2 Systemtap
Systemtap 提供了如下了 tapset 进行网络追踪:
1. Socket Tapset
2. Networking Tapset

## 2.网络跟踪
### 2.1 套接字连接
由处理网络的应用程序函数，系统套接字库，系统调用层，或者在内核中都可以跟踪套接字活动。通常偏好在系统调用层，因为文档最全，系统开销最低并且是系统级的。

除了 connect() 和 accept() 还能跟踪 socket() 和 close() 系统调用，这允许在创建时发现文件描述符，并用时间差衡量套接字的持续时间。

#### Dtrace
```bash
# 1. 用 connect() 计数出站连接
dtrace -n 'syscall::connect:entry { @[execname] =count(); }'

# 2. 用 accept() 计算入站连接
dtrace -n 'syscall::accept:entry { @[execname] =count(); }'

# 3. 通过检查用户栈可以揭示为什么会执行套接字
dtrace -n 'syscall::connect:entry /execname == 'ssh'/ { ustack(); }'

# 4. 检查系统调用的参数，源自 Gregg
# https://github.com/brendangregg/bpf-perf-tools-book
soconnect.d  
```

#### Systemtap
```bash
# 1. 用 connect() 计数出站连接
stap -ve 'global s;probe syscall.connect { s[execname()] <<< 1}
                   probe end { foreach(i in s- limit 10)
                             {printf("%s: %d\n", i, @count(s[i]))}}'

# 2. 用 accept() 计算入站连接
stap -ve 'global s;probe syscall.accept { s[execname()] <<< 1}
                   probe end { foreach(i in s- limit 10)
                             {printf("%s: %d\n", i, @count(s[i]))}}'
                             
# 3. 通过检查用户栈可以揭示为什么会执行套接字
stap -ve 'global s; probe syscall.connect { s[ubacktrace()] <<< 1}
                    probe end { foreach(i in s- limit 10)
                              {print_ustack(i); printf("%d\n", @count(s[i]))}}'

# 4. 检查系统调用的参数，源自 Gregg
# https://github.com/brendangregg/bpf-perf-tools-book
soconnect.d 

# bpftrace 安装
sudo yum install epel-release
sudo yum install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
sudo snap install --devmode bpftrace
sudo snap connect bpftrace:system-trace

# 执行 soconnect.dt
git clone https://github.com/brendangregg/bpf-perf-tools-book.git
cd bpf-perf-tools-book/originals/Ch10_Networking/
bpftrace soconnect.bt
```

### 2.2 套接字I/O
#### Dtrace
```bash
# 1. 按 execname 计数套接字读取次数
dtrace -n 'syscall::read:entry,syscall::recv:entry
           /fds[arg0].fi_fs == "sockfs"/ { @[execname]=count() }'

# 2. 按 execname 计数套接字读取次数
dtrace -n 'syscall::write:entry,syscall::send:entry
           /fds[arg0].fi_fs == "sockfs"/ { @[execname]=count() }'
```

#### Systemtap
socket tapset 提供套接字读取的详细信息
```bash
# 1. 按 execname 计数套接字读取次数
stap -ve 'global s; probe socket.receive { s[execname()] <<< 1 }
                     probe end { foreach(i in s- limit 10)
                               { printf("%s: %d\n", i, @count(s[i]))} }'

# 2. 按 execname 计数套接字读取次数
stap -ve 'global s; probe socket.send  { s[execname()] <<< 1 }
                     probe end { foreach(i in s- limit 10)
                               { printf("%s: %d\n", i, @count(s[i]))} }'
```
### 2.3 套接字延时
延时包括:
1. 连接延时:
    - 同步系统调用: 就是 connect() 耗时
    - 异步系统调用: 这是执行 connect() 至 poll() 或者 select() 或其他系统调用报告套接字就绪的时间
2. 首字节延时: 自执行 connect() 或从 accept() 返回，直到第一字节数据由任何一个 I/O 系统调用从套接字接收到的时间
3. 套接字持续时间: 同一个套接字从 socket() 到 close() 的时间要聚焦连接时长可以由connect() 或者 accetp() 开始计时

#### Dtrace
```bash
```

#### Systemtap
```bash
```

### 2.4 套接字内部活动
#### Dtrace
利用 fbt provider 能跟踪套接字的内核运行。
```bash
# linux 查看套接字内核函数探针
dtrace -ln 'fbt::sock*:entry'
```

#### Systemtap
```bash
# linux 查看套接字内核函数探针
> stap -l 'kernel.function("sock*")'
kernel.function("sock_aio_dtor@net/socket.c:890")
kernel.function("sock_aio_read@net/socket.c:959")
kernel.function("sock_aio_write@net/socket.c:1001")
kernel.function("sock_alloc@net/socket.c:535")
.....

> stap -L 'kernel.function("sock_recv_drops@net/socket.c:788")'
kernel.function("sock_recv_drops@net/socket.c:788") $skb:struct sk_buff* $sk:struct sock* $msg:struct msghdr*
```

### 2.5 TCP 事件
#### Dtrace
TCP 的内核运行也能用 fbt provider，但是 tcp 有专属的 tcp provider
|TCP 探针|描述|
|:---|:---|
|tcp:::accept-established|接受一个入站连接，被动 open|
|tcp:::connect-request|启动一个出站连接，主动 open|
|tcp:::connect-established|建立一个出站连接，完成三次握手|
|tcp:::accetp-refused|拒绝一个连接请求，关闭本地端口|
|tcp:::connect-refused|拒绝一个连接请求，关闭远程端口|
|tcp:::send|发送一个数据段，ip可能直接将数据段映射到一个数据包|
|tcp:::receive|接受一个数据段，ip可能直接将数据段映射到一个数据包|
|tcp:::state-change|一个会话发生状态改变|

tcp provider 提供了协议包头细节以及内核内部状态，其中包含"缓冲"的进程ID，通常DTrace 内置的 execname 跟踪的进程名不一定有效，因为内核 TCP 时间可能与进程非同步发生。

```bash
# 以频率计数接受的 TCP 连接(被动)以及远程 IP 地址和本地端口
dtrace -n 'tcp:::accept-established{ @[args[3]->tcps_raddr, args[3]->tcps_lport] = count(); }'
```

#### Systemtap
```bash
# 以频率计数接受的 TCP 连接(被动)以及远程 IP 地址和本地端口
stap -ve 'global s; probe kernel.{function("tcp_accept"),function("inet_csk_accept")}.return?
                    {sock = $return;if (sock != 0){s[inet_get_local_port(sock), inet_get_ip_source(sock)] <<< 1}}
                    probe end { foreach([i,j] in s- limit 10)
                               { printf("%s,%d: %d\n", j,i, @count(s[i,j]))} }'
```

tcp 连接监控
```bash
#! /usr/bin/env stap

probe begin {
  printf("%6s %16s %6s %6s %16s\n",
         "UID", "CMD", "PID", "PORT", "IP_SOURCE")
}

probe kernel.{function("tcp_accept"),function("inet_csk_accept")}.return? {
  sock = $return
  if (sock != 0)
    printf("%6d %16s %6d %6d %16s\n", uid(), execname(), pid(),
           inet_get_local_port(sock), inet_get_ip_source(sock))
}
```

### 2.6 数据包传输
tcp provider 提供的功能有限，跟踪内核函数是跟踪网络传输的根本方法。一个快速穿越网络栈的方法是跟踪一个深层次的事件并且检查它的调用栈。  
#### Dtrace
```bash
# 1. 跟踪网络调用内核栈
> dtrace -n 'fbt::ip_output:entry { @[stack(100)] = count(); }'
.....                      
kernel`tcp_sendmsg_Ox895`  # 网络调用内核栈中的一个函数
.....

# 2. 跟踪调用栈中的特定系统调用比如 tcp_sendmsg，查看他的参数
# 假设 tcp_sendmsg 的第四个参数是以字节为单位的长度，我们就可以统计 TCP 发送段的长度
> dtrace -n 'fbt::tcp_sendmsg:entry { @['TCP send bytes']=quantize(arg3); }'

```

#### Systemtap
```bash
```

### 2.7 重传跟踪
研究 TCP 重传有助于调查网络健康程度
#### Dtrace
```bash
tcp_retransmit_skb.d

# 云计算性能优化工具包中用于跟踪重传的 dtrace 脚本
# https://github.com/brendangregg/dtrace-cloud-tools
tcpretranssnoop.d
```

#### Systemtap
```bash
git clone https://github.com/brendangregg/systemtap-lwtools.git
# 未找到类似工具
```

### 2.8 积压队列丢包
#### Dtrace
```bash
# 云计算性能优化工具包中用于跟踪 积压队列丢包的 dtrace 脚本
# https://github.com/brendangregg/dtrace-cloud-tools
tcpconnreqmaxq-pid_sdc6.d
```

#### Systemtap
```bash
# 未找到类似工具
```

### 3. 高级网络跟踪脚本
1. Dtrace
    - [云计算性能优化工具包](https://github.com/brendangregg/dtrace-cloud-tools)
    - [DTraceBook 的 Network Lower-Level Protocals 章节](http://www.brendangregg.com/dtracebook/index.html)
    - https://github.com/brendangregg/bpf-perf-tools-book
2. Systemtap
    - https://sourceware.org/systemtap/SystemTap_Beginners_Guide/useful-systemtap-scripts.html#mainsect-network
    - https://github.com/brendangregg/bpf-perf-tools-book
    - https://github.com/brendangregg/systemtap-lwtools.git
