# 4.3 文件管理命令


bash常见特性

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->


本节我们将学习与文件目录管理相关的命令，包括
1. 目录管理类命令
2. 文件查看类命令
3. 文件管理工具

## 1. 目录管理类的命令：
#### mkdir
`mkdir [OPTION]... DIRECTORY...`
- 作用: 创建目录
- 选项:
	- `-p`: 父目录不存在时，自动按需创建父目录；
	- `-v`: verbose，显示详细过程；
	- `-m`: MODE：直接给定权限；

#### rmdir
`rmdir [OPTION]... DIRECTORY...`
- 作用: 删除空目录
- 选项:
	- `-p`：删除某目录后，如果其父目录为空，则一并删除之；
	- `-v`: 显示过程；

### tree：
`tree [options] [directory]`
- 作用: 以层级方式展开显示目录
- 选项:
	- `-L level`：指定要显示的层级；


## 2. 文件查看类命令：
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

#### more
`more FILE`
- 作用: 分屏查看文件内容，没有 less 常用，了解即可
- 特点：翻屏至文件尾部后自动退出；

#### less
`less FILE`
- 作用: 分屏查看文件内容, man 手册页即是调用 less 命令显示的结果
- 可用的操作如下
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


#### head
`head [options] FILE`
- 作用: 查看文件的前n行；
- 选项:					
	- `-n #`: 指定查看多少行
	- `-#`: 同上，`#` 表示数字

#### tail
tail [options] FILE
- 作用: 查看文件的后n行；
- 选项:
	- `-n #`: 指定查看多少行
	- `-#`: 同上，`#` 表示数字
	- `-f`: 查看文件尾部内容结束后不退出，跟随显示新增的行；

#### stat
`stat FILE...`
- 作用: display file or file system status
```
> stat SUMMARY.md
  文件："SUMMARY.md"
  大小：16876     	块：40         IO 块：4096   普通文件
设备：fd03h/64771d	Inode：603785      硬链接：1
权限：(0664/-rw-rw-r--)  Uid：( 1000/     tao)   Gid：( 1000/     tao)
环境：unconfined_u:object_r:user_home_t:s0
最近访问：2018-07-06 09:04:02.197198385 +0800  # access time
最近更改：2018-07-06 09:04:02.185198436 +0800  # modify time 最近修改文件内容的时间
最近改动：2018-07-06 09:04:02.185198436 +0800  # change time 最近文件元数据发生更改的时间
创建时间：-
```

#### touch：
`touch [OPTION]... FILE...`
- 作用: 更改文件的时间戳
- 选项:
	- `-c`: 指定的文件路径不存在时不予创建；
	- `-a`: 仅修改access time；
	- `-m`: 仅修改modify time；
	- `-t`: STAMP 格式为 `[[CC]YY]MMDDhhmm[.ss]`


### 3. 文件管理工具
#### copy
**命令式使用**:
1. 单源复制：`cp [OPTION]... [-T] SOURCE DEST`
	- 如果DEST不存在：则事先创建此文件，并复制源文件的数据流至DEST中；
	- 如果DEST存在：
		- 如果DEST是非目录文件：则覆盖目标文件；
		- 如果DEST是目录文件：则先在DEST目录下创建一个与源文件同名的文件，并复制其数据流；
2. 多源复制：
	- `cp [OPTION]... SOURCE... DIRECTORY`
	- `cp [OPTION]... -t DIRECTORY SOURCE...`
	- 如果DEST不存在：错误；
	- 如果DEST存在：
		- 如果DEST是非目录文件：错误；
		- 如果DEST是目录文件：分别复制每个文件至目标目录中，并保持原名；

**常用选项**：
- `-i`：交互式复制，即覆盖之前提醒用户确认；
- `-f`：强制覆盖目标文件；
- `-r, -R`：递归复制目录；
- `-d`：== `--no-dereference --preserve=links`复制符号链接文件本身，而非其指向的源文件；
- `-a, -- archive`：相当于`-dR --preserve=all`，用于实现归档；
- `--preserv=`:保留源文件哪些属性，可用值如下
	- `mode`：权限
	- `ownership`：属主和属组
	- `timestamps`: 时间戳
	- `context`：安全标签
	- `xattr`：扩展属性
	- `links`：符号链接
	- `all`：上述所有属性

#### mv
`mv [OPTION]... [-T] SOURCE DEST`  
`mv [OPTION]... SOURCE... DIRECTORY`  
`mv [OPTION]... -t DIRECTORY SOURCE..`    	
- 使用: 同 `cp`
- 作用: 移动文件或目录
- 选项：
	- `-i`：交互式；
	- `-f`：force

#### rm
`rm [OPTION]... FILE...`
- 作用: 删除文件或目录
- 选项：
	- `-i`：interactive，交互
	- `-f`：force，强制删除
	- `-r`: recursive，递归删除
- 删除目录：`rm -rf /PATH/TO/DIR`
- 危险操作：`rm -rf /*`
- 注意：所有不用的文件建议不要直接删除，而是移动至某个专用目录；（模拟回收站）


#### tr
`tr [OPTION]... SET1 [SET2]`
- 作用: 把输入的数据当中的字符，凡是在SET1定义范围内出现的，通过对位转换为SET2出现的字符
- 用法1：`tr SET1 SET2 < /PATH/FROM/SOMEFILE`  -- 用 SET2 替换 SET1
- 用法2：`tr -d SET1 < /PATH/FROM/SOMEFILE`    -- 删除 SET 1中出现的字符
- 注意：不修改原文件

```
# 把/etc/passwd文件的前6行的信息转换为大写字符后输出；
> head -n 6 /etc/passwd | tr 'a-z' 'A-Z'


> tr [a-z]  [A-Z]
how are you
HOW ARE YOU
```

