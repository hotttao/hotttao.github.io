# 4.6 内存动态追踪


本节我们来介绍内存动态追踪技术
<!-- more -->

## 1. Systemtap 进行 内存 分析
Systemtap 可以用来剖析
1. 跟踪用户和内核的内存分配
2. 主次缺页异常
3. 页面换出守护进程的运行

这些功能支持负载特征分析、向下挖掘分析。

### 1.1 分配跟踪
用户级别的分配跟踪使用 pid provider，内核的分配跟踪使用 fbt provider
#### Dtrace
```bash
# 1. 按进程汇报用户级 malloc 请求的长度
# arg0 参数记录了 malloc 请求的字节数
dtrace -n 'pid$target::malloc:entry { @["request"] = quantize(arg0); }' -p PID
dtrace -n 'pid$target::malloc:entry { @[ustack()] = quantize(arg0); }' -p PID

# 2. 计算libumem函数调用
# 列出libumem分配器的入口探针
dtrace -ln 'pid$target:libumem::entry' -p PID
dtrace -n 'pid$target:libumem::entry { @[probefunc] = count(); }' -p PID

# 3. 按堆增长(通过 brk())计算用户栈
dtrace -n 'syscall::brk:entry { @[execname, ustack()] = count(); }'

# 4. 按照缓存名称和栈跟踪内核 slab 分配器
dtrace -n 'fbt::kmem_cache_alloc:entry' { @[stringof(arg[0]->cache_name), stack()] = count(); }
```


#### Systemtap
```bash
# 1. 汇报用户级 malloc 请求的长度
stap -ve 'global s; probe vm.kmalloc { s[execname(),caller_function] <<< bytes_req}
                    probe end { foreach([i,j] in s){printf("%s %s: %d\n", i, j, @count(s[i,j]))}}'
# 3. 
stap -ve 'global s; probe vm.brk { s[ubacktrace()] <<< 1}
                    probe end { foreach(i in s- limit 10)
                              {print_ustack(i); printf("%d\n", @count(s[i]))}}'
# 4. 
stap -ve 'global s; probe vm.kmem_cache_alloc { s[caller_function] <<< 1; }
                    probe end { foreach(i in s- limit 10)
                              {printf("%s: %d\n", i, @count(s[i]))}}'
```

### 1.2 缺页跟踪
跟踪缺页异常能更深入解释系统如何分配内存，可以利用 fbt provider，或者在可用的情况下使用稳定的 vminfo provider
#### Dtrace
```bash
# 1.跟踪 beam.smp 进程的缺页异常
# as_fault 次缺页异常，maj_fault 主缺页异常
dtrace -n 'vminfo:::as_fault /execname == 'beam.smp' / { @[ustack(4)] =  count();}'

# 2. 匿名页面换入探测, 跟踪整个系统，按频率统计引起匿名页面换入进程ID和进程名
dtrace -n 'vminfo:::anonpgin { @[pid, execname] =  count();}'
```

#### Systemtap
```bash
# 1. 跟踪系统的主缺页异常
stap -ve 'global s; probe vm.pagefault.return { if (fault_type == VM_FAULT_MAJOR){s[execname()] <<< 1}}
                    probe end { foreach(i in s- limit 10){printf("%s: %d\n", i, @count(s[i]))}}'
# 2. 跟踪整个系统，按频率统计引起匿名页面
stap -ve 'global s; probe vm.pagefault.return { if (fault_type == VM_FAULT_SIGBUS){s[execname()] <<< 1}}
                    probe end { foreach(i in s- limit 10){printf("%s: %d\n", i, @count(s[i]))}}'
```

