# 11.2 网络接口命名与配置指南


网络接口命名与配置指南

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

上一节我们讲解了网络的基础知识，很推荐大家读一读[计算机网络-自顶向下方法](https://book.douban.com/subject/26176870/)。本节我们会介绍 Linux 中网络接口(网卡)的命名方式，以及概括性的说一说 Linux 中进行网络配置的方式；在接下来的章节中，我们会详细讲解每个命令的使用。本节内容如下:
1. 网络配置方式
2. Linxu 网络接口的命名方式

## 1. 网络配置方式
将一台 Linux 主机接入到网络中，需要为其配置如下几个参数
1. IP/NETMASK：本地通信
2. 路由（网关）：跨网络通信
3. DNS服务器地址：基于主机名的通信

这些参数的配置可以通过命令直接修改内核中的网络参数，也可以修改配置文件然后让内核重载配置文件或下次重新启动生效；也可以依赖本地局域网中配置的 DHCP 服务，为局域网中的其他主机动态配置。DHCP(Dynamic Host Configure Procotol) 服务的配置我们会在后面的章节中介绍。Linux 中配置网络属性的命令和相关配置文件如下

### 1.1 网络属性管理
网络属性配置的配置文件不同的发行版有所不同，RedHat及相关发行版的配置文件位于 `/etc/sysconfig/network-scripts/ifcfg-NETCARD_NAME`，其中 NETCARD_NAME 为特定网络接口的名称。网络属性管理有众多命令家族，概述如下:
1. ifcfg家族：
    - `ifconfig`：配置IP，NETMASK
    - `route`：路由查看与管理
    - `netstat`：网络状态及统计数据查看
2. iproute2家族：
    - `ip OBJECT`：ip 命令下有众多子命令
        - `addr`：管理和查看地址和掩码；
        - `link`：网络接口属性管理
        - `route`：路由查看与管理
    - `ss`：网络状态及统计数据查看
3. CentOS 7特有的 nm(Network Manager)家族
    - `nmcli`：nm 命令行工具
    - `nmtui`：text window 工具
4. Centos6 特有的:
    - `system-config-network-tui`
    - `setup`, `setup` 拥有专属的配置文件 `system-config-netword-tui`

### 1.2 DNS服务
#### DNS 服务配置
DNS 服务只能通过修改其配置文件 `/etc/resolv.conf` 进行配置。Linux 主机最多可指定三个 DNS 服务器

```bash
# vim /etc/resolv.conf
nameserver 10.143.22.116  # 主DNS服务器地址
nameserver 10.143.22.118  # 备用DNS服务器地址
nameserver 10.143.22.116  # 第三备份DNS服务器地址
```

#### DNS 服务测试
测试 DNS 服务是否正常，可以使用 `host`, `nslookup`, `dig` 三个命令
1. 正解: FQDN(域名)到 IP
    - `dig  -t  A  FQDN`
    - `host -t A FQND`
2. 反解: IP 到 FQDN
    - `dig  -x  IP`
    - `host -t PTR IP`

DNS 是互联网的基础服务，我们会花一整章节，来详细介绍 DNS 服务。

### 1.3 本地主机名配置
主机名有三种配置方式 `hostname`, `hostnamectl` 和修改配置文件

#### hostname
- 查看：`hostname`
- 配置：`hostname  HOSTNAME`
- 效力：只对当前系统有效，重启后无效；

#### hostnamectl
Centos7 新增的特有命令
- `hostnamectl  status`：显示当前主机名信息；
- `hostnamectl  set-hostname`：设定主机名，永久有效；
- 效力: 通过 hostnamectl 修改的主机名立即生效，且永久有效

#### 修改配置文件
- 主机名的配置文件位于 `/etc/sysconfig/network`
- 效力：此方法的设置不会立即生效； 但以后会一直有效；

```
# vim /etc/sysconfig/network
HOSTNAME="hostname"
```

## 3. 网络接口命名方式
网络接口(网卡)的命名在Linux 中有特定设置过程。默认情况下，Centos6 采用传统的命名机制，Centos7 采用可预测命名方案，支持多种不同的命名机制，这种命名机制需要 biosdevname 程序的参与

### 3.1 命名机制
1. Centos6 传统命名：
    - 以太网：`ethX, [0,oo)`，例如eth0, eth1, ...
    - PPP网络：`pppX, [0,...]`, 例如，ppp0, ppp1, ...    
2. CentOS7 可预测命名方案：支持多种不同的命名机制
    1. 如果Firmware(固件)或BIOS为主板上集成的设备提供的索引信息可用，则根据此索引进行命名，如eno1, eno2, ...
    2. 如果Firmware或BIOS为PCI-E扩展槽所提供的索引信息可用，且可预测，则根据此索引进行命名，如ens1, ens2, ...
    3. 如果硬件接口的物理位置信息可用(硬件接口的拓扑结构)，则根据此信息命名，如enp2s0, ...
    4. 如果用户显式定义根据MAC地址命名，例如enx122161ab2e10, ...
    5. 上述均不可用，则仍使用传统方式命名；

### 3.2 名称组成格式  
Centos7 中 eno1，ens1，enp2s0 命名组成如下所示:
1. 前缀
    - `en`：ethernet 以太网接口
    - `wl`：wlan 无线局域网设备
    - `ww`：wwan 无线广域网设备
2. 后缀
    - `o<index>`: 集成设备的设备索引号；(onbus)
    - `s<slot>`: 扩展槽的索引号；
    - `x`: 基于MAC地址的命名；
    - `p<bus>s<slot>`: 基于总线及槽的拓扑结构进行命名；
        - `bus`: PCI 总线编号
        - `slot`: 总线上的扩展槽编号

### 3.3 网卡设备的命名过程
Centos7 网卡命名经过了以下过程:
1. udev 辅助工具程序 `/lib/udev/rename_device` 会根据 `/usr/lib/udev/rules.d/60-net.rules` 中的指示去查询  `/etc/sysconfig/network-script/ifcfg-IFACE` 配置文件，根据HWADDR 读取设备名称
2. biosdevname 根据 `/user/lib/udev/rules.d/71-boosdevname.rules`
3. 通过检查网络接口设备，根据 `/usr/lib/udev/rules.d/75-net-description` 中 `ID_NET_NAME_ONBOARD` 和 `ID_NET_NAME_SLOT`,`ID_NET_NAME_PATH 命名`

Centos7 也可以设置网络接口回归传统方式的命名方式:
1. `vim /etc/default/grub` 配置文件，添加 `GRUB_CMDLINE_LINUX="net.ifnames=0 rhgb quiet"`
2. 为 grub2 生成配置文件 `grub2-mkconfig -o /etc/grub2.cfg`
3. 重启系统

