# 4.10 磁盘


本节我们来介绍磁盘相关的操作系统原理。磁盘I/O可能会造成严重性能的问题(注意是可能)。在高负载下，磁盘成为瓶颈，CPU 持续空转以等待磁盘磁盘I/O结束。
<!-- more -->

## 1. 磁盘相关的操作原理
我们将从下面几个方面入手来讲解磁盘相关的操作系统原理:
1. 磁盘的基础知识
2. 磁盘I/O栈

最后我们会说一说磁盘检测的相关指标。

## 2. 磁盘的基础知识
这一部分内容与磁盘的构造相关，能帮助解释为什么磁盘慢。目前我们使用的磁盘主要有两种类型:
1. 磁性旋转机械盘，也就是我们常说的机械硬盘
2. 基于闪存的 SSD

### 2.1 机器硬盘
机械硬盘(hard disk drive HDD) 由机械手臂，磁头，盘片组成。慢I/O通常由**磁头寻道时间**,**盘片旋转时间**造成。

#### 磁盘缓存
这些磁盘共有的一个部件是一小块内存(RAM)用来缓存读取的结果和缓冲要写入的数据。还允许I/O命令在设备上排队，以更高效的方式重新排序。

#### 电梯寻道
电梯算法又名电梯寻道是提高命令队列效率的一种方式。它根据磁盘位置把I/O重新排序，最小化磁头的移动。电梯算法会容易对偏移量远的I/O操作造成饥饿。

#### ECC
磁盘在每个扇区的结尾存储了一个纠错码，以便在数据读取时进行验证并有可能纠错。如果验证失败，可能发生重读，这可能是异常缓慢I/O的原因。检查操作系统和磁盘上的错误计数器以确认。


### 2.2 SSD
固态磁盘（Solid State Disk），通常缩写为 SSD，由固态电子元器件组成。固态磁盘不需要磁道寻址，所以，不管是连续 I/O，还是随机 I/O 的性能，都比机械磁盘要好得多。

固态磁盘来说，虽然它的随机性能比机械硬盘好很多，但同样存在“先擦除再写入”的限制。随机读写会导致大量的垃圾回收，所以相对应的，随机 I/O 的性能比起连续 I/O 来，也还是差了很多。

## 3. 磁盘I/O栈
磁盘I/O栈的组件和层次取决于操作系统、版本和采用的软硬件技术。下面延时了一个通用模型:

![disk_io](/images/linux_pf/disk_io.png)
- Block Device Interface: 块设备接口
- Buffer Cache: 缓冲区高速缓存
- Target I/O Driver: 目标I/O驱动
- Multpathing I/O Driver: 多路I/O驱动
- Host Bus Adaptor Driver: 主机总线适配器

### 3.1 Linux的块层
Linux 中，磁盘实际上是作为一个块设备来管理的。每个块设备都会被赋予两个设备号，分别是主、次设备号。主设备号用在驱动程序中，用来区分设备类型；而次设备号则是用来给多个同类设备编号。

块I/O设备一般可以通过 iostat 监控。Linux 改进了内核组成了块层。通用块层，其实是处在文件系统和磁盘驱动中间的一个块设备抽象层，主要有两个作用:
1. 第一个功能跟虚拟文件系统的功能类似。向上，为文件系统和应用程序，提供访问块设备的标准接口；向下，把各种异构的磁盘设备抽象为统一的块设备，并提供统一框架来管理这些设备的驱动程序。
2. 第二个功能 I/O调度器: 通用块层会给文件系统和应用程序发来的 I/O 请求排队，并通过重新排序、请求合并等方式，提高磁盘读写的效率。

下面是 Linux 块层的示意图:

![block_layer](/images/linux_pf/block_layer.png)
- Virtual Block Driver: 虚拟快驱动
- Elevator Layer: 电梯层
- I/O Scheduler: I/O 调度器
- Physical Block Driver: 物理块驱动

电梯层提供了通用功能，例如排序，合并以及聚合请求发送。

I/O调度器使 I/O 能够排队排序或者重新调度以优化发送，具体由调度策略决定，可用的策略如下:
1. 空操作: 不调度
2. 截止时间: 试图强制给延迟设置截止时间
3. 预期: 通过启发式方法预测I/O
4. CFQ: 完全公平调度器

### 3.2 I/O 栈
结合文件系统和今天的内容，我们可以把 Linux 存储系统的 I/O 栈，由上到下分为三个层次:
1. 文件系统层，包括虚拟文件系统和其他各种文件系统的具体实现。它为上层的应用程序，提供标准的文件访问接口；对下会通过通用块层，来存储和管理磁盘数据
2. 通用块层，包括块设备 I/O 队列和 I/O 调度器。它会对文件系统和应用程序的 I/O 请求进行排队，再通过重新排序和请求合并，然后才要发送给下一级的设备层。
3. 设备层，包括存储设备和相应的驱动程序，负责最终物理设备的 I/O 操作。

## 4.磁盘检测
下面是磁盘常用的性能测量指标:
1. IOPS
2. 使用率
3. 饱和度

IOPS 很难横向比较，有意义的IOPS需要包含其他细节:
1. 随机I/O还是连续I/O
2. I/O大小
3. 读写比例

基于时间的指标使用率和饱和度可以更简单的进行比较。

在介绍这些指标之前，我们先来看看磁盘性能领域的一些重要概念：测量时间


### 4.1 测量时间
存储设备的响应时间指的是从I/O请求到结束的时间，由服务和等待时间组成:
1. 等待时间: I/O在队列中等待服务的时间
2. 服务时间: I/O 得到处理的时间

![disk_io_time](/images/linux_pf/disk_io_time.png)

响应时间，服务时间，等待时间取决于测量所处的位置。在上面的磁盘io 栈的示意图中，io 栈的每一层都有可能实现自己的队列，在不同的测量位置可以得到不同的等待时间和服务时间。iostat 展示的磁盘设备接口的服务时间只是一种简化。

### 4.2 使用率
使用率通过某段时间内磁盘运行时间的忙时间的比例计算得出。为了确定高使用率是否会导致应用程序性能问题，需要研究磁盘的反映时间和应用程序是否阻塞在此I/O上。

### 4.3 饱和度
饱和度可以通过操作系统的磁盘等待队列的长度计算得出。

### 4.4 磁盘I/O vs 应用程序I/O
最后犹如文件系统一章所提到的，文件系统性能比磁盘性能更加重要。最快的I/O就是没有I/O，所以要充分利用缓存来降低磁盘I/O的次数。

由于经过了中间的层层组件，磁盘I/O与应用程序I/O在频率和大小上都不匹配。所以需要细致研究磁盘I/O与应用程序阻塞之间的关系。而不能仅仅通过数值的大小去评判。

