# 21.4 网络防火墙配置


网络防火墙配置

![linux-mt](/images/linux_mt/linux_iptables.jpg)
<!-- more -->

前面我们讲解了 iptables 命令的使用，其中主要是以配置主机防火墙作为示例，本节将来介绍如何配置一个网络防火墙。iptables 命令的使用没变，只是网络防火墙配置载 forward 链上有一些额外注意的事项。

# 1. 网络防火墙配置
在进行网络防火墙配置之前，我们首先需要规划一下网络拓扑结构，好方便解说 iptables 命令的作用。

![network](/images/linux_mt/net_filter.png)

如上图所示，左半部分是我们模拟的内网，右半部分是模拟的公网，在使用 virtualbox 或 vimware 模拟上述网络时，有以下几点需要注意:
1. 内网的网卡类型选择仅主机或 nat 网络，外网的网卡选择桥接。有关虚拟网卡的几种类型请参考 [24.7 虚拟网卡类型](24-iptables/虚拟网卡类型.md)
2. 为使得从外网 `192.168.1.10` 返回的响应能到达我们的内网，需要将其网关指向 `192.168.1.168`,或者手动添加路由条目将，发往 `172.16.0.0/24` 的报文的下一跳设置成 `192.168.1.168`
3. 需要打开中间的网络防火墙的核心转发功能
4. 为测试防火墙，配置，我们需要在 `172.16.0.2` 和 `192.168.1.10` 上均配置好 httpd,vsftpd 等服务

```
# 添加路由
route add -net 172.16.0.0/24 gw 192.168.1.168

# 打开ia核心转发
sysctl -w net.ipv4.ip_forward=1
```

## 1.1 放行 httpd
防火墙 filter 功能只能添加在 `INPUT FORWARD OUTPUT` 链上，对于网络防火墙而言，报文只会经过 `FORWARD` 链，因此网络防火墙只能配置在 `FORWARD` 链上。一次 http 事务包括请求和响应两个过程，因此我们需要在 `FORWARD` 上同时添加发送请求和接收响应两个方向的规则。下面是配置示例，我们的目的是，内网的主机能访问所有的外网主机，但外网主机仅能访问内网的 httpd,ftp 服务。

```
modprobe nf_conntrack_ftp
# 设置默认策略为拒绝
$ iptables -A FORWARD -j DROP

# 开放 80 端口
$ iptables -I FORWARD -s 172.16.0.0/24 -p tcp --dport 80 -j ACCEPT
$ iptables -I FORWARD 2 -d 172.16.0.0/24 -p tcp --sport 80 -j ACCEPT

# 开放 ftp
$ iptables -R FORWARD 3 -s 172.16.0.0/24 -p tcp -m state --stat RELATED -j ACCEPT
$ iptables -R FORWARD 4 -p tcp -d 172.16.0.0/24 -m state --state ESTABLISHED  -j ACCEPT
```

## 1.2 基于连接追踪机制配置
使用连接追踪机制，可以让 iptables 规则更加简单
```
# 开放内网到外网的所有请求
$ iptables -A FORWARD -j DROP

# 1. 开放已经建立的连接
$ iptables -R FORWARD 1 -p tcp -m state --state ESTABLISHED -j ACCEPT

# 2. 开放由内到外的新连接,此时 ftp 也可访问，因为 RELATED 也是 NEW 连接
$ iptables -I FORWARD 2 -p tcp -s 172.16.0.0/24 -m state --state NEW -j ACCEPT

# 开放外网到内网特定主机的 80 访问
$ iptables -I FORWARD 2 -d 172.16.0.10 -p tcp --dport 80 -m state --state NEW -j ACCEPT
```


## 2. 利用 iptables 抵御 DOS 攻击
利用iptables的recent模块来抵御DOS攻击:
1. 建立一个列表，保存有所有访问过指定的服务的客户端IP
2. ssh: 远程连接，

```
# one
> iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 3 -j DROP

# two
> iptables -I INPUT -p tcp --dport 22 -m state --state NEW -m recent --set --name SSH

# three
> iptables -I INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 300 --hitcount 3 --name SSH -j LOG --log-prefix "SSH Attach: "

# four
> iptables -I INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 300 --hitcount 3 --name SSH -j DROP

# 也可以使用下面的这句记录日志：
> iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --name SSH --second 300 --hitcount 3 -j LOG --log-prefix "SSH Attack"
```

1. one:
2. two:
- 利用connlimit模块将单IP的并发设置为3；会误杀使用NAT上网的用户，可以根据实际情况增大该值；
    - 第二句是记录访问tcp 22端口的新连接，记录名称为SSH
    - --set 记录数据包的来源IP，如果IP已经存在将更新已经存在的条目
3. three
    - 利用recent和state模块限制单IP在300s内只能与本机建立2个新连接。被限制五分钟后即可恢复访问。
4. four:
    - 第三句是指SSH记录中的IP，300s内发起超过3次连接则拒绝此IP的连接。
    - --update 是指每次建立连接都更新列表；
    - --seconds必须与--rcheck或者--update同时使用
    - --hitcount必须与--rcheck或者--update同时使用
5. 附注:
    - iptables的记录：/proc/net/xt_recent/SSH

