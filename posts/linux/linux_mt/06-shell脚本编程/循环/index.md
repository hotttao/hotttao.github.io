# 6.3 循环


循环

![linux-mt](/images/linux_mt/linux_mt.jpg)
<!-- more -->

本节我们来学习 bash shell 编程的第三部分循环，包括以下内容:
1. for 循环
2. while 循环
3. until 循环
3. 循环体内的控制语句 continue, break

## 1. for 循环
for 循环通过遍历列表的方式执行循环，列表生成有如下几种方式

### 1.1 列表生成方式
- 直接给出:`user1 user2 user3`
- 整数列表
    - `{start..end}`: 使用内置关键字`{}` 生成整数列表
    - `$(seq [start  [incremtal]] last)`: 使用 seq 命令生成整数列表
- 返回列表的命令，例如 `ls` 命令
- glob 通配符，例如 `for file in /var/*; do; done`
- 变量引用，例如 `$*`, `$@`

### 1.2 for 循环语法
for 循环的语法，及其使用示例如下所示
```bash
for  VARAIBLE  in  LIST; do
    循环体
done
```

```bash
#!/bin/bash
#
declare -i uphosts=0
declare -i downhosts=0

for i in {1..17}; do
    if ping -W 1 -c 1 172.16.$i.1 &> /dev/null; then
        echo "172.16.$i.1 is up."
        let uphosts+=1
    else
        echo "172.16.$i.1 is down."
        let downhosts+=1
    fi
done
echo "Up hosts: $uphosts, Down hosts: $downhosts."
```


### 1.3 类 C 风格for 循环
for 循环除通常的列表遍历外，还有一种类 C 风格使用方法，其语法如下
- 控制变量初始化：仅在循环代码开始运行时执行一次；
- 控制变量的修正语句：每轮循环结束会先进行控制变量修正运算，而后再做条件判断；

```bash
for  ((控制变量初始化;条件判断表达式;控制变量的修正语句)); do
    循环体
done
```

```bash
# 示例：求100以内所有正整数之和
#!/bin/bash
#
declare -i sum=0

for ((i=1;i<=100;i++)); do
    let sum+=$i
done

echo "Sum: $sum."
```



## 2. while 循环
### 2.1 while 条件循环
while 循环只要条件满足，就会一直执行
```bash
while  CONDITION1; do
    循环体
done
```

```bash
#!/bin/bash
#
declare -i i=1
declare uphosts=0
declare downhosts=0
net="172.169.250"

while [ $i -le 20 ]; do
    if ping -c 1 -w $net.$i &>/dev/null; then
        echo "$net.$i is up"
        let uphosts++
    fi
done
```

### 2.2 while 文件遍历
while循环还有一种特殊用法，可用于文件遍历。如下所示 while 将依次读取`/PATH/FROM/SOMEFILE` 文件中的每一行，且将其赋值给VARIABLE变量
```bash
while  read  VARIABLE; do
    循环体；
done  <  /PATH/FROM/SOMEFILE
```

```bash
# 示例：找出ID号为偶数的用户，显示其用户名、ID及默认shell；
#!/bin/bash
#
while read line; do
    userid=$(echo $line | cut -d: -f3)
    username=$(echo $line | cut -d: -f1)
    usershell=$(echo $line | cut -d: -f7)

    if [ $[$userid%2] -eq 0 ]; then
        echo "$username, $userid, $usershell."
    fi
done < /etc/passwd                
```

### 2.3 创建死循环
`while true` 可以创建死循环， `sleep` 命令可以让进程休眠一段时间

`sleep NUMBER[SUFFIX]`:
- 作用: 让程序在 sleep 处休眠 NUMBER 秒
- SUFFIX: 默认为 s, 指暂停的秒数, m 指分钟, h 指小时, d 代表天数

```bash
#!/bin/bash
# 练习：每隔3秒钟到系统上获取已经登录用户的用户的信息；其中，如果logstash用户登录了系统，则记录于日志中，并退出；

while true; do
    if who | grep "^logstash\>" &> /dev/null; then
        break
    fi
    sleep 3
done

echo "$(date +"%F %T") logstash logged on" >> /tmp/users.log    
```

## 3. until 循环
until 循环只要条件满足，就会退出循环

```bash
until  CONDITION; do
  循环体
done
```

```bash
#!/bin/bash
# 练习：每隔3秒钟到系统上获取已经登录用户的用户的信息；其中，如果logstash用户登录了系统，则记录于日志中，并退出；
until who | grep "^logstash\>" &> /dev/null; do
    sleep 3
done

echo "$(date +"%F %T") logstash logged on" >> /tmp/users.log        
```

### 4. 循环控制语句
### 4.1 continue
continue 可提前结束本轮循环，直接进入下一轮循环判断；

```bash            
# 示例：求100以内所有偶数之和；                                        
#!/bin/bash
#
declare -i evensum=0
declare -i i=0

while [ $i -le 100 ]; do
    let i++
    if [ $[$i%2] -eq 1 ]; then
        continue
    fi
    let evensum+=$i
done

echo "Even sum: $evensum"
```                

### 3.2 break
break 可提前跳出整个循环。在下面的示例中 `while true` 将创建死循环，达到满足的条件时，break 将跳出循环。

```bash
# 示例：求100以内所奇数之和
#!/bin/bash
#
declare -i oddsum=0
declare -i i=1

while true; do
    let oddsum+=$i
    let i+=2
    if [ $i -gt 100 ]; then
        break
    fi
done
```

