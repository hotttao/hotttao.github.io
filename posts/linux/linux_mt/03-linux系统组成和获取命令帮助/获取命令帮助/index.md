# 3.2 Linux 命令帮助


Linux 中获取命令帮助

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

通过之前的学习我们了解到 Linux 命令分为两类，一类是 shell 内置的内嵌命令，另一类是外部命令。这两种命令获取帮助的并不相同，对于内部命令使用 `help command` 即可。而对于外部命令有很多方式，其中最重要也是最便捷的就是使用 man 帮助手册。本节我们将学习如何获取外部命令的使用帮助，以及了解 man 手册的使用方式。

## 1. 获取命令的使用帮助
内部命令：`help COMMAND`  
外部命令：
1. `COMMAND --help`: 命令自带简要格式的使用帮助
2.  `man COMMAND`: 使用手册：manual
3. `info COMMAND`: 获取命令的在线文档；
4. 很多应用程序会自带帮助文档：`/usr/share/doc/APP-VERSION`，包括不限于
	- README：程序的相关的信息；
	- INSTALL: 安装帮助；
	- CHANGES：版本迭代时的改动信息；
5. 主流发行版官方文档: eg：http://www.redhat.com/doc
6. 程序官方的文档：官方站点上的"Document"
7. 搜索引擎

```
# google 搜索引擎的使用技巧
keyword filetype:pdf
keyword site:domain.tld
```

## 2. man 帮助手册的使用简述
### 2.1 man 手册页组成
关于man的简介如下列表所示:
1. man手册的原始文档位于 `/usr/share/man`中，
2. 使用手册为压缩格式的文件，有章节之分,包括 `ls /usr/share/man`
	- man1：用户命令；
	- man2：系统调用；
	- man3：C库调用；
	- man4：设备文件及特殊文件；
	- man5：文件格式；（配置文件格式）
	- man6：游戏使用帮助；
	- man7：杂项；
	- man8：管理工具及守护进行；
3. 当我们 `man COMMAND` 进入man手册后，其由如下几个部分组成。界面示例如下
	- SECTION：组成部分
		- `NAME`：功能性说明
		- `SYNOPSIS`：语法格式
		- `DESCRIPTION`：描述
		- `OPTIONS`：选项
		- `EXAMPLES`：使用示例
		- `AUTHOR`: 作者
		- `BUGS`: 报告程序bug的方式
		- `SEE ALSO`: 参考
	- SYNOPSIS: 命令使用
		- `[]`：可选内容；
		- `<>`：必须提供的内容；
		- `a|b|c`：多选一；
		- `...`：同类内容可出现多个；

```
> man ifconfig
IFCONFIG(8)            Linux System Administrator's Manual           IFCONFIG(8)
NAME
       ifconfig - configure a network interface

SYNOPSIS
       ifconfig [-v] [-a] [-s] [interface]
       ifconfig [-v] interface [aftype] options | address ...

NOTE
       This  program  is  obsolete!   For replacement check ip addr and ip link.
       For statistics use ip -s link.

DESCRIPTION
       Ifconfig is used to configure the kernel-resident network interfaces.  It
       is  used  at boot time to set up interfaces as necessary.  After that, it
       is usually only needed when debugging or when system tuning is needed.
.............
```


### 1.2 man 命令使用
`man [CHAPTER] COMMAND`
- 作用: 在特定章节中搜索命令的帮助手册, CHAPTER 参数可选
- 注意：
	- 并非每个COMMAND在所有章节下都有手册；
	- 可通过 `whatis COMMAND` 查看哪些章节中存在COMMAND 的帮助手册
	- `whatis`的执行过程是查询数据库进行的，必要时可通过 `makewhatis` 手动更新数据库


### 1.2 man 手册查看操作
进入man手册页之后，其界面环境就是调用 less 命令的执行结果，可用的操作如下
1. 翻屏：
	- `空格键`：向文件尾翻一屏；
	- `b`: 向文件首部翻一屏；
	- `Ctrl+d`：向文件尾部翻半屏；
	- `Ctrl+u`：向文件首部翻半屏；
	- `回车键`：向文件尾部翻一行；
	- `k`: 向文件首部翻一行；
	- `G`：跳转至最后一行；
	- `#G`: 跳转至指定行；
	- `1G`：跳转至文件首部；
2. 文本搜索：
	- `/keyword`：从文件首部向文件尾部依次查找；不区分字符大小写；
	- `?keyword`：从文件尾部向文件首部依次查找；
	- `n`: 与查找命令方向相同；
	- `N`: 与查找命令方向相反；
3. 退出：`q(quit)`

