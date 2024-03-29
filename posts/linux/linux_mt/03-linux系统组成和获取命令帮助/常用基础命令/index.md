# 3.1 Linux 命令基础


Linux 基础命令

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->


在安装完 Centos 之后，我们开始正式学习 Linux。在深入学习 Linux 之前，我们需要学习一些最基本命令的使用。我们将分成两节来学习下面内容:
1. Linux 中的命令类型
2. Linux 命令的标准使用格式
1. Linux 常见基础命令的使用
2. 如何使用 Linux 帮助文档

Linux 各个部分大多都有标准进行规范，以降低在不同发行版之间的迁移难度，命令也不例外。本节我们首先将学习命令的语法通用格式，然后了解 Linu 中命令的分类；最后介绍 Linux 常用的基础命令。

## 1. 命令的语法通用格式：
`COMMAND OPTIONS ARGUMENTS`
- COMMAND:
	- 命令名，发起一命令，请求内核将某个二进制程序运行为一个进程；
	- 命令本身是一个可执行的程序文件：二进制格式的文件，有可能会调用共享库文件；
	- 多数系统程序文件都存放在：`/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`，`/usr/local/bin`, `/usr/local/sbin`
		- 普通命令：`/bin`, `/usr/bin`, `/usr/local/bin`
		- 管理命令：`/sbin`, `/usr/sbin`, `/usr/local/sbin`
		- 共享库：`/lib`, `/lib64`, `/usr/lib`, `/usr/lib64`, `/usr/local/lib`, `/usr/local/lib64`
			- 32bits的库：`/lib`, `/usr/lib`, `/usr/local/lib`
			- 64bits的库：`/lib64`, `/usr/lib64`, `/usr/local/lib64`
	- 注意：并非所有的命令都有一个在某目录与之对应的可执行程序文件
	- 命令必须遵循特定格式规范：exe, msi, ELF(Linux)
	- 查看命令类型: `file /bin/ls`
- OPTIONS：
	- 作用: 命令选项，指定命令的运行特性；
	- 类型: 选项有两种表现形式：
		- 短选项：-C, 例如-l, -d
			- 注意：有些命令的选项没有-；
			- 如果同一命令同时使用多个短选项，多数可合并：-l -d = -ld
		- 长选项：--word, 例如--help, --human-readable
			- 注意：长选项不能合并；
	- 注意：有些选项可以带参数，此称为选项参数；
- ARGUMENTS：
	- 命令的作用对象；命令对什么生效；
	- 注意：不同的命令的参数；有些命令可同时带多个参数，多个之间以空白字符分隔；
- 例如：ls -ld /var /etc


## 2. Linxu 的命令类型
- 命令分为两类：
	1. 内置命令(builtin): 由shell程序的自带的命令：
	2. 外部命令: 独立的可执行程序文件，文件名即命令名：
- 查看命令类型：`type COMMAND`
	- 内部命令显示为 builtin
	- 外部命令显示为命令文件路径；
	- 注意：命令可以有别名；别名可以与原名相同，此时原名被隐藏；此时如果要运行原命令，则使用\COMMAND；


## 3. Linxu 的常用命令
### 3.1 文件与目录查看命令
#### pwd:
- 作用: 显示工作目录

#### cd：
`cd [/PATH/TO/SOMEDIR]`
- 作用: change directory 改变当前工作目录
- 选项:
	- `cd`: 切换回家目录；注意：
	- `cd ~`：切换回自己的家目录，bash中, ~表示家目录；
	- `cd ~USERNAME`：切换至指定用户的家目录；
	- `cd -`：在上一次所在目录与当前目录之间来回切换；
- 附注: 相关的环境变量
	- `$PWD`：当前工作目录
	- `$OLDPWD`：上一次的工作目录


#### ls
`ls [OPTION] [FILE]`
- 作用: list, 列出指定目录下的内容
- 选项:
	- `-a`: 显示所有文件，包括隐藏文件；
	- `-A`: 显示除.和..之外的所有文件；
	- `-l`: --long, 长格式列表，即显示文件的详细属性信息；
	- `-h`: --human-readable：对文件大小单位换算；换算后结果可能会非精确值；
	- `-d`: 查看目录自身而非其内部的文件列表；
	- `-r`: reverse, 逆序显示；
	- `-R`: **recursive，递归显示**；

#### cat：
`cat [OPTION] [FILE]`
- 作用: concatenate, 文件文本查看工具；
- 选项:				
	- `-n`：给显示的文本行编号；
	- `-E`: 显示行结束符$；

#### tac：
`tac [OPTION] [FILE]`
- 作用: 文件文本查看工具；
- 选项:				
	- `-n`：给显示的文本行编号；
	- `-E`: 显示行结束符$；

#### file
`file [FILE]`
- 作用: 查看文件内容类型；


#### echo
`echo [SHORT-OPTION] [STRING]`
- 作用: 回显
- 选项: 	
	- `-n`: 不进行换行；
	- `-e`：让转义符生效；

**Linxu 中引号的效力**
- STRING可以使用引号，单引号和双引号均可用；
- 单引号：强引用，变量引用不执行替换；
- 双引号：弱引用，变量引用会被替换；
- 附注: 变量引用可使用 `${name}` 和 `$name`


### 3.2 关机与重启命令
#### shutdown
`shutdown [OPTIONS] [TIME] [WALL]`
- 作用: 关机或重启命令
- OPTIONS:
	- `-h`: halt
	- `-r`: reboot
	- `-c`: cancel
- TIME：
	- now: 立刻马上
	- hh:mm: 指定几时几秒
	- +m: m 分钟后
	- +0


### 3.3 日期相关的命令：
Linux 系统启动时从硬件读取日期和时间信息；读取完成以后，就不再与硬件相关联；此时系统时钟与硬件时钟是相互独立的

#### date
`date [OPTION] [+FORMAT]`
- 作用: 显示系统时钟日期时间：
- 选项:
	- `-d String|-d @timestamp`: 显示指定的时间字符串或时间戳表示的时间
	- `-s, --set=STRING`: 设置系统时间
- FORMAT：格式符
	- `%F`:= `%Y-%m-%d`
	- `%T`:直接显示时间 (24 小时制)
	- `%Y`:完整年份 (0000-9999)
	- `%m`:月份 (01-12)
	- `%d`:日 (01-31)
	- `%H`:小时(00-23)
	- `%M`:分钟(00-59)
	- `%S`:秒(00-60)
	- `%s`:从1970年1月1号(unix元年)0点0分0秒到命令执行那一刻经过的秒数；

```
# 按特定格式显示时间
> date +%s
1531134611

> date -d @1531134611 +%F
2018-07-09

> date -d "2017-10-13 23:00:00" +%F
2017-10-13

> date

# 设置系统时间
> date -s "2018-10-13 23:10:12"
> date -s "2018/10/13 23:10:12"
```
#### hwclock / clock
`hwclock [function] [option]`
- 作用:显示或设定硬件时钟
- 选项:
	- `-s`: --hctosys,以硬件为准，把系统调整为与硬件时间相同；
	- `-w`: --systohc：以系统为准，把硬件时间调整为与系统时钟相同；				

#### cal：
`cal [[month] year]`
- 作用: 显示日历

### 3.4 命令别名与查找
`alias [alias-name[=string]`
- 作用: 查看或定义命令别名：
- 常用:
	- `alias`: 获取所有可用别名的定义：
	- `alias NAME='COMMAND'`：
		- 定义别名：
		- 注意：仅对当前shell进程有效
	- `unalias NAME`: 撤销别名：

#### which
`which [options] programname`
- 作用: shows the full path of (shell) commands
- 选项:
	- `--skip-alias`：忽略别名

#### whereis命令：
`whereis [options] name`
- 作用: locate the binary, source, and manual page files for a command
- 选项:
	- `-b`: 仅搜索二进制程序路径；
	- `-m`：仅搜索使用手册文件路径；


### 3.5 登录用户查看
#### who
`who [OPTION]`
- 作用: show who is logged on
- 选项:
	- `-b`: 系统此次启动的时间；
	- `-r`: 显示系统运行级别；

#### w
`w [user]`
- 作用: Show who is logged on and what they are doing. 增强版的 who

#### tty
`tty`
- 作用: 显示当前终端

