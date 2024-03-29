# 2.6 eBPF

今天我们来讲第二动态追踪技术 eBPF，eBPF 就是 Linux 版的 DTrace，可以通过 C 语言自由扩展。
<!-- more -->

## 1. eBPF 简介
eBPF 的工作原理如下图所示:

![eBPF_arch](/images/linux_pf/eBPF_arch.png)

 eBPF 通过 C 语言自由扩展（这些扩展通过 LLVM 转换为 BPF 字节码后，加载到内核中执行。从图中你可以看到，eBPF 的执行需要三步：
 1. 从用户跟踪程序生成 BPF 字节码；
 2. 加载到内核中运行；
 3. 向用户空间输出结果。

 实际上，在 eBPF 执行过程中，编译、加载还有 maps 等操作，对所有的跟踪程序来说都是通用的。把这些过程通过 Python 抽象起来，也就诞生了 BCC（BPF Compiler Collection）。

 BCC 把 eBPF 中的各种事件源（比如 kprobe、uprobe、tracepoint 等）和数据操作（称为 Maps），也都转换成了 Python 接口（也支持 lua）。这样，使用 BCC 进行动态追踪时，编写简单的脚本就可以了。

 不过要注意，因为需要跟内核中的数据结构交互，真正核心的事件处理逻辑，还是需要我们用 C 语言来编写。

### 1.1 bcc 安装
```bash
# REHL 7.6
$ yum install bcc-tools

# 安装后，BCC 会把所有示例（包括 Python 和 lua），放到 /usr/share/bcc/examples 目录中
$ ls /usr/share/bcc/
exampleshello_world.py  lua  networking  tracing
```

### 1.2 eBPF 与 Systemtap
在 eBPF 出现之前，SystemTap 是 Linux 系统中，功能最接近 DTrace 的动态追踪机制。SystemTap 在很长时间以来都游离于内核之外（而 eBPF 自诞生以来，一直根植在内核中。从稳定性上来说，SystemTap 只在 RHEL 系统中好用，在其他系统中则容易出现各种异常问题。当然，反过来说，支持 3.x 等旧版本的内核，也是 SystemTap 相对于 eBPF 的一个巨大优势。


## 2. eBPF 编码示例
接下来，以 do_sys_open 为例，我们一起来看看，如何用 eBPF 和 BCC 来动态跟踪 do_sys_open 系统调用。

如下面的代码所示，通常我们可以把 BCC 应用，拆分为下面这四个步骤。
1. 第一，跟所有的 Python 模块使用方法一样，在使用之前，先导入要用到的模块 BPF
2. 第二，需要定义事件以及处理事件的函数。这个函数需要用 C 语言来编写，作用是初始化刚才导入的 BPF 对象。这些用 C 语言编写的处理函数，要以字符串的形式送到 BPF 模块中处理：
3. 定义一个输出函数，并把输出函数跟 BPF 事件绑定
4. 最后一步，就是执行事件循环，开始追踪 do_sys_open 的调用

```python
# 1. 导入模块
from bcc import BPF

# 2. 定义事件以及处理事件的函数
# define BPF program (""" is used for multi-line string).
# '#' indicates comments for python, while '//' indicates comments for C.
prog = """
#include <uapi/linux/ptrace.h>
#include <uapi/linux/limits.h>
#include <linux/sched.h>
// define output data structure in C
struct data_t {
    u32 pid;
    u64 ts;
    char comm[TASK_COMM_LEN];
    char fname[NAME_MAX];
};
BPF_PERF_OUTPUT(events);

// define the handler for do_sys_open.
// ctx is required, while other params depends on traced function.
int hello(struct pt_regs *ctx, int dfd, const char __user *filename, int flags){
    struct data_t data = {};
    data.pid = bpf_get_current_pid_tgid();
    data.ts = bpf_ktime_get_ns();
    if (bpf_get_current_comm(&data.comm, sizeof(data.comm)) == 0) {
        bpf_probe_read(&data.fname, sizeof(data.fname), (void *)filename);
    }
    events.perf_submit(ctx, &data, sizeof(data));
    return 0;
}
"""
# load BPF program
b = BPF(text=prog)
# attach the kprobe for do_sys_open, and set handler to hello
b.attach_kprobe(event="do_sys_open", fn_name="hello")

# 3. 定义一个输出函数，并把输出函数跟 BPF 事件绑定
# process event
start = 0
def print_event(cpu, data, size):
    global start
    # event’s type is data_t
    event = b["events"].event(data)
    if start == 0:
            start = event.ts
    time_s = (float(event.ts - start)) / 1000000000
    print("%-18.9f %-16s %-6d %-16s" % (time_s, event.comm, event.pid, event.fname))

# loop with callback to print_event
b["events"].open_perf_buffer(print_event)

# 4. 就是执行事件循环，开始追踪 do_sys_open 的调用
# print header
print("%-18s %-16s %-6s %-16s" % ("TIME(s)", "COMM", "PID", "FILE"))
# start the event polling loop
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```


通过这个简单的示例，你也可以发现，eBPF 和 BCC 的使用，其实比 ftrace 和 perf 有更高的门槛。想用 BCC 开发自己的动态跟踪程序，至少要熟悉 C 语言、Python 语言、被跟踪事件或函数的特征（比如内核函数的参数和返回格式）以及 eBPF 提供的各种数据操作方法。

BCC 软件包也内置了很多已经开发好的实用工具，默认安装到 /usr/share/bcc/tools/ 目录中，后面我们会详细介绍各个工具的使用。


