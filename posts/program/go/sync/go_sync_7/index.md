# Cond 条件变量


Cond 条件变量
<!-- more -->

## 1. Cond 概述
Go 标准库提供 Cond 原语的目的是，为**等待 / 通知**场景下的并发问题提供支持。Cond 通常应用于等待某个条件的一组 goroutine，等条件变为 true 的时候，其中一个 goroutine 或者所有的 goroutine 都会被唤醒执行。顾名思义，Cond 是和某个条件相关:
1. 条件没有满足时，所有等待这个条件的 goroutine 都被阻塞
2. 条件满足时，等待的 goroutine 可以继续进行执行

顾名思义，Cond 是和某个条件相关，而使用 Cond 的是两组协程:
1. 第一组 goroutine 共同协作完成条件
2. 第二组 goroutine 等待直至条件完成

Cond 在实际项目中被使用的机会比较少，原因总结起来有两个:
1. 因为一旦遇到需要使用 Cond 的场景，我们更多地会使用 Channel 的方式去实现，因为 channel 才是更地道的 Go 语言的写法
2. 对于简单的 wait/notify 场景，比如等待一组 goroutine 完成之后继续执行余下的代码，我们会使用 WaitGroup 来实现

但是 Cond 有三点特性是 Channel 无法替代的：
1. Cond 和一个 Locker 关联，可以利用这个 Locker 对相关的依赖条件更改提供保护
2. Cond 可以同时支持 Signal 和 Broadcast 方法，而 Channel 只能同时支持其中一种
3. Cond 的 Broadcast 方法可以被重复调用。等待条件再次变成不满足的状态后，我们又可以调用 Broadcast 再次唤醒等待的 goroutine。这也是 Channel 不能支持的，Channel 被 close 掉了之后不支持再 open

本质上 WaitGroup 和 Cond 是有区别的:
1. WaitGroup 是主 goroutine 等待确定数量的子 goroutine 完成任务；
2.  Cond 是等待某个条件满足，这个条件的修改可以被任意多的 goroutine 更新，而且 Cond 的 Wait 不关心也不知道其他 goroutine 的数量，只关心等待条件
3. 而且 Cond 还有单个通知的机制，也就是 Signal 方法

### 1.1 Cond 使用
标准库中的 Cond 并发原语初始化的时候，需要关联一个 Locker 接口的实例，一般我们使用 Mutex 或者 RWMutex。Cond 初始化和提供的方法如下:

```go
type Cond
  func NeWCond(l Locker) *Cond
  func (c *Cond) Broadcast()
  func (c *Cond) Signal()
  func (c *Cond) Wait()
```

1. 首先，Cond 关联的 Locker 实例可以通过 c.L 访问，它内部维护着一个先入先出的等待队列
2. Signal 方法:
    - 允许调用者 Caller 唤醒一个等待此 Cond 的 goroutine
    - Cond 等待队列中有多个等待的 goroutine 时，需要从等待队列中移除第一个 goroutine 并唤醒
    - 调用 Signal 方法时，不强求你一定要持有 c.L 的锁
3. Broadcast 方法:
    - 允许调用者 Caller 唤醒所有等待此 Cond 的 goroutine
    - 如果 Cond 等待队列中有一个或者多个等待的 goroutine，则清空所有等待的 goroutine，并全部唤醒
    - 调用 Broadcast 方法时，也不强求你一定持有 c.L 的锁
4. Wait 方法: 
    - 把调用者 Caller 放入 Cond 的等待队列中并阻塞
    - 调用 Wait 方法时必须要持有 c.L 的锁
    - 至于为什么调用 Wait() 必须要持有锁，我的理解是，**所有调用 Wait() 的方法都需要检查条件是否满足，甚至会改变检查条件，它们彼此应该是互斥的，需要使用锁保护检查条件**

下面是 Cond 的使用示例:

```go

func main() {
    c := sync.NewCond(&sync.Mutex{})
    var ready int

    for i := 0; i < 10; i++ {
        go func(i int) {
            time.Sleep(time.Duration(rand.Int63n(10)) * time.Second)

            // 加锁更改等待条件
            // 注意点一: 条件变量的更改，其实是需要原子操作或者互斥锁保护的
            c.L.Lock()
            ready++
            c.L.Unlock()

            log.Printf("运动员#%d 已准备就绪\n", i)
            // 广播唤醒所有的等待者
            c.Broadcast()
        }(i)
    }

    // 注意点三: Wait() 方法调用前需要先获取锁
    c.L.Lock()
    // 注意点二: Wait 唤醒后需要检查条件
    // 我们一定要记住，waiter goroutine 被唤醒不等于等待条件被满足
    for ready != 10 {
        c.Wait()
        log.Println("裁判员被唤醒一次")
    }
    c.L.Unlock()

    //所有的运动员是否就绪
    log.Println("所有运动员都准备就绪。比赛开始，3，2，1, ......")
}
```

## 2. Cond 实现
Cond 的实现非常简单，或者说复杂的逻辑已经被 Locker 或者 runtime 的等待队列实现了。

```go

type Cond struct {
    noCopy noCopy

    // 当观察或者修改等待条件的时候需要加锁
    L Locker

    // 等待队列
    notify  notifyList
    checker copyChecker
}

func NewCond(l Locker) *Cond {
    return &Cond{L: l}
}

func (c *Cond) Wait() {
    c.checker.check()
    // 增加到等待队列中
    t := runtime_notifyListAdd(&c.notify)
    // 把当前调用者加入到 notify 队列之中后会释放锁
    c.L.Unlock()
    // 阻塞休眠直到被唤醒
    // 等调用者被唤醒之后，又会去争抢这把锁
    runtime_notifyListWait(&c.notify, t)
    c.L.Lock()
}

func (c *Cond) Signal() {
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify)
}

func (c *Cond) Broadcast() {
    c.checker.check()
    runtime_notifyListNotifyAll(&c.notify）
}
```

在 Cond 实现中:
1. runtime_notifyListXXX 是运行时实现的方法，实现了一个等待 / 通知的队列，代码位于  runtime/sema.go 中
2. copyChecker 是一个辅助结构，可以在运行时检查 Cond 是否被复制使用
3. Signal 和 Broadcast 只涉及到 notifyList 数据结构，不涉及到锁
4. Wait 把调用者加入到等待队列时会释放锁，在被唤醒之后还会请求锁。在阻塞休眠期间，调用者是不持有锁的，这样能让其他 goroutine 有机会检查或者更新等待变量。

### 2.1 开源项目的应用
开源项目中使用 sync.Cond 的代码少之又少，Kubernetes 中有一个使用 Cond 的例子。Kubernetes 项目中定义了优先级队列 PriorityQueue 这样一个数据结构，用来实现 Pod 的调用。它内部有三个 Pod 的队列，即 activeQ、podBackoffQ 和 unschedulableQ，其中 activeQ 就是用来调度的活跃队列（heap）。Pop 方法调用的时候，如果这个队列为空，并且这个队列没有 Close 的话，会调用 Cond 的 Wait 方法等待。

```go

// 从队列中取出一个元素
func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
    p.lock.Lock()
    defer p.lock.Unlock()
    for p.activeQ.Len() == 0 { // 如果队列为空
      if p.closed {
        return nil, fmt.Errorf(queueClosed)
      }
      p.cond.Wait() // 等待，直到被唤醒
    }
    ......
    return pInfo, err
  }

```

当 activeQ 增加新的元素时，会调用条件变量的 Boradcast 方法，通知被 Pop 阻塞的调用者。

```go

// 增加元素到队列中
func (p *PriorityQueue) Add(pod *v1.Pod) error {
    p.lock.Lock()
    defer p.lock.Unlock()
    pInfo := p.newQueuedPodInfo(pod)
    if err := p.activeQ.Add(pInfo); err != nil {//增加元素到队列中
      klog.Errorf("Error adding pod %v to the scheduling queue: %v", nsNameForPod(pod), err)
      return err
    }
    ......
    p.cond.Broadcast() //通知其它等待的goroutine，队列中有元素了

    return nil
  }
```

这个优先级队列被关闭的时候，也会调用 Broadcast 方法，避免被 Pop 阻塞的调用者永远 hang 住。

```go

func (p *PriorityQueue) Close() {
    p.lock.Lock()
    defer p.lock.Unlock()
    close(p.stop)
    p.closed = true
    p.cond.Broadcast() //关闭时通知等待的goroutine，避免它们永远等待
}
```

对于需要重复调用 Broadcast 的场景，比如这里的 Kubernetes 的例子，每次往队列中成功增加了元素后就需要调用 Broadcast 通知所有的等待者，使用 Cond 就再合适不过了。

## 3. Cond 采坑点
使用 Cond 时有两个常见错误
1. 一个是调用 Wait 的时候没有加锁
2. 另一个是没有检查条件是否满足程序就继续执行了

我们一定要记住，waiter goroutine 被唤醒不等于等待条件被满足，只是有 goroutine 把它唤醒了而已，等待条件有可能已经满足了，也有可能不满足，我们需要进一步检查。你也可以理解为，等待者被唤醒，只是得到了一次检查的机会而已。

## 4. Cond 使用上的理解
Cond 在使用 Wait() 方法前需要调用 Lock() 这一点，在使用上的确很容易让人迷惑，我个人的理解是这样的。Cond 的核心是对条件的判断，改变条件相当于写，是需要加锁的，那同时**判断条件是否满足是读，同样也需要加锁**。调用 Wait 方法前，肯定已经判断了条件不满足，此时必定是加锁了。所以在 Wait 方法内，先释放锁，唤醒后在加锁，是读条件必须加锁这个场景要求的。Wait 释放锁再加锁是果，而不是因，我们要牢记的是**在读写条件的时候都必须加锁**。于此同时也正是因为 Cond 的条件是在实现之外维护的，所以 Cond 支持条件比 Channel 和 WaitGroup 更加灵活。

## 参考
本文内容摘录自:
1. [极客专栏-鸟叔的 Go 并发编程实战](https://time.geekbang.org/column/intro/100061801?tab=catalog)
