# 6.7 程序交互与信号捕捉


程序交互，退出状态与信号捕捉

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

本节我们开始学习 bash shell 编程的第七部分，包含以下内容:
1. 如何在程序执行过程中与用户交互
2. bash 编程调试
3. 设置脚本执行的状态返回值
4. 如何在 bash 中使用 ASCII 颜色
5. dialog 实现窗口化编程
6. 信号捕捉


## 1. 用户交互
bash 中与用户交互主要通过 read 命令完成，read 可以输出一段文字提示用户输入指定内容，并将用户输入保存在特定变量中，read 的使用方式如下

`read [OPTIONS] [变量名 ...]`
- 作用:从标准输入读取一行并将其分为不同的域
- 选项:
  - `-p 提示符`: 用户提示符
  - `-t 超时`: 设置等待用户输入的超时时长，单位秒
  - `-a 数组`
  - `-d 分隔符`
  - `-i 缓冲区文字`
  - `-n 读取字符数`
  - `-N 读取字符数`
  - `-u 文件描述符`

```
> read -p "enter a disk filename"  -t  5  name
> [  -z  "$name"  ]   &&  name="value"

> read a b
jerry mark cd

> echo $a
jerry

> echo $b  # 值个数多于变量个数时，所有多余的变量默认保存在最后一个变量中
mark cd
```

## 2. bash 调试
bash 命令有两个参数可以帮助我们调试
- `bash  -n  script.sh`  -- 检查脚本语法错误
- `bash  -x  script.sh`  --  单步执行，显示代码执行的详细过程


## 3. 脚本的状态返回值
之前我们说过，程序执行有状态返回值，保存在 `$?` 变量中。shell 脚本的状态返回值有如下特点:
1. 默认是脚本中执行的最后一条件命令的状态返回值
2. 使用 `exit [n]` 可自定义退出状态码，n 位于 0-255 之间，0 表示执行成功无异常，默认为 0
3. exit 类似 python 中的return，程序遇到 exit 即终止

## 4. 在bash中使用ACSII颜色
`\033[31m hello \033[0m`
- 格式: `\033[31m` 表示颜色控制开始符，`\033[0m` 表示颜色控制结束符，中间为要显示的文本
- 颜色控制: `##m`(`31m`)
		- 左侧`#`：
      - 3：表示前景色
			- 4：表示背景色
		- 右侧 `#`: 表示颜色种类, 范围是 1, 2, 3, 4, 5, 6, 7
- 格式控制: `#m`(`5m`) 加粗、闪烁等功能；
- 组合: 多种控制符，可组合使用，彼此间用分号隔开；

```bash
tao@hp:shell$ echo -e "\033[31m htto \033[0m"
 htto

tao@hp:shell$ echo -e "\033[41;32m htto \033[0m"
 htto

tao@hp:shell$ echo -e "\033[42;35;4m htto \033[0m"
htto
```

## 5. dialog 实现窗口化编程

```
yum install dialog -y

dialog --msgbox  hello 17 30
```

本节我们来学习 bash shell 编程的第九部分如何在 shell 中捕捉信号。shell 中捕捉信号主要是使用 shell 内置的 trap 命令。shell 中的信号捕捉有以下几点需要特别注意。

`15) SIGTERM` 和 `9) SIGKILL` 信号是不可捕捉的，以防止不能终止进程。

bash 中的命令会以子进程运行，因此信号可能会被子进程捕获，执行脚本的进程因此可能无法捕获到信号。但是如果 trap 在子命令之前优先执行，信号则会优先被执行脚本的进程捕获。

程序执行过程中可能会产生很多临时文件或其他数据，正常结束时，这些临时文件都会被清理；但是如果程序执行过程中被 Ctrl-C 终止可能这些临时数据将无法被清除；因此可能需要捕捉 `2) SIGINT` 信号以清除执行程序时产生的临时文件。

## 6. 信号捕捉
#### trap
`trap -l`:
- 作用: 等同于 `kill -l` 列出所有信号

`trap  'COMMAND'  SIGNALS`
- 作用: 指定在接收到信号后将要采取的动作，常见的用途是在脚本程序被中断时完成清理工作
- 常可以进行捕捉的信号：
  - `1) SIGHUP`
  - `2) SIGINT`

```bash
# 表示当shell收到HUP INT PIPE QUIT TERM这几个命令时，当前执行的程序会读取参数“exit 1”，并将它作为命令执行。
trap "exit 1" HUP INT PIPE QUIT TERM
```

#### 示例
```bash
#!/bin/bash
#
declare -a hosttmpfiles
trap  'mytrap'  INT

mytrap()  {
  echo "Quit"
  rm -f ${hosttmpfiles[@]}
  exit 1
}


for i in {1..50}; do
  tmpfile=$(mktemp /tmp/ping.XXXXXX)
  if ping -W 1 -c 1 172.16.$i.1 &> /dev/null; then
    echo "172.16.$i.1 is up" | tee $tmpfile
  else
    echo "172.16.$i.1 is down" | tee $tmpfile
  fi
  hosttmpfiles[${#hosttmpfiles[*]}]=$tmpfile
done

rm -f ${hosttmpfiles[@]}
```

