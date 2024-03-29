# go 分组操作的并发控制


## 1. 分组操作概述
共享资源保护、任务编排和消息传递是 Go 并发编程中常见的场景，而分组执行一批相同的或类似的任务则是任务编排中一类情形，本节我们来介绍一下**分组编排**的一些常见场景和并发原语，包括:
1. ErrGroup
2. gollback
3. Hunch
4. schedgroup

这些分组执行的并发原语相比于 WaitGroup 目的是为了增加获取子任务的错误信息，以及子任务的并发数。

## 2. ErrGroup
[ErrGroup](https://github.com/golang/sync/tree/master/errgroup):
- 包位置: `golang.org/x/sync/errgroup`
- 适用场景: 将一个通用的父任务拆成几个小任务并发执行的场景
- 底层实现: 基于 WaitGroup
- 提供功能:
    - 和 Context 集成
    - error 向上传播，可以把子任务的错误传递给 Wait 的调用者

### 2.1 ErrGroup 实现

```go
package errgroup

import (
	"context"
	"sync"
)

// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid and does not cancel on error.
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once
	err     error
}

// WithContext returns a new Group and an associated Context derived from ctx.
//
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
func WithContext(ctx context.Context) (*Group, context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{cancel: cancel}, ctx
}

// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
	g.wg.Wait()
	if g.cancel != nil {
		g.cancel()
	}
	return g.err
}

// Go calls the given function in a new goroutine.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()

		if err := f(); err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
	}()
}
```

ErrGroup 有三个方法分别是 WithContext、Go 和 Wait。

WithContext:
- 作用: 创建一个 Group 对象
- 签名: `func WithContext(ctx context.Context) (*Group, context.Context)`
- 返回: Group 实例和使用 context.WithCancel(ctx) 生成的新 Context，一旦有一个子任务返回错误，或者 Wait 调用返回，新的 Context 就会被 cancel
- 注意:
    - Group 的零值也是合法的，只不过，你就没有一个可以监控是否 cancel 的 Context 了
    - 如果传递给 WithContext 的 ctx 参数，是一个可以 cancel 的 Context 的话，那么，它被 cancel 的时候，并不会终止正在执行的子任务

Go 方法:
- 作用: 执行子任务
- 签名: `func (g *Group) Go(f func() error)`
- 执行: 子任务函数 f 是类型为 func() error 的函数，如果任务执行成功，就返回 nil，否则就返回 error，并且会 cancel 那个新的 Context
- 注意: 
    - 可能有多个子任务执行失败返回 error，Wait 方法只会返回第一个错误，所以，如果想返回所有的错误，需要特别的处理
    - 处理方式是使用全局的 result slice 保存子任务的执行结果

Wait:
- 作用: 所有的子任务都完成后，它才会返回，否则只会阻塞等待
- 签名: `func (g *Group) Wait() error`
- 返回: 如果有多个子任务返回错误，它只会返回第一个出现的错误，如果所有的子任务都执行成功，就返回 nil
- 说明: 要获取所有任务返回的 error，可以使用一个 result slice 保存子任务的执行结果

### 2.2 使用示例
使用 20 goroutine 计算传入目录下所有文件的 md5 值:

```go
package main

import (
	"context"
	"crypto/md5"
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"path/filepath"

	"golang.org/x/sync/errgroup"
)

// Pipeline demonstrates the use of a Group to implement a multi-stage
// pipeline: a version of the MD5All function with bounded parallelism from
// https://blog.golang.org/pipelines.
func main() {
	m, err := MD5All(context.Background(), ".")
	if err != nil {
		log.Fatal(err)
	}

	for k, sum := range m {
		fmt.Printf("%s:\t%x\n", k, sum)
	}
}

type result struct {
	path string
	sum  [md5.Size]byte
}

// MD5All reads all the files in the file tree rooted at root and returns a map
// from file path to the MD5 sum of the file's contents. If the directory walk
// fails or any read operation fails, MD5All returns an error.
func MD5All(ctx context.Context, root string) (map[string][md5.Size]byte, error) {
	// ctx is canceled when g.Wait() returns. When this version of MD5All returns
	// - even in case of error! - we know that all of the goroutines have finished
	// and the memory they were using can be garbage-collected.
	g, ctx := errgroup.WithContext(ctx)
	paths := make(chan string) // 文件路径channel

    // 1. 遍历目录下的所有文件，发送到 channel paths 中
	g.Go(func() error {
		defer close(paths)
		return filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return err
			}
			if !info.Mode().IsRegular() {
				return nil
			}
			select {
			case paths <- path:
			case <-ctx.Done():
				return ctx.Err()
			}
			return nil
		})
	})

	// 启动20个goroutine执行计算md5的任务，计算的文件由上一阶段的文件遍历子任务生成.
	c := make(chan result)
	const numDigesters = 20
	for i := 0; i < numDigesters; i++ {
		g.Go(func() error {
			for path := range paths {
				data, err := ioutil.ReadFile(path)
				if err != nil {
					return err
				}
				select {
				case c <- result{path, md5.Sum(data)}:
				case <-ctx.Done():
					return ctx.Err()
				}
			}
			return nil
		})
	}

    
	go func() {
		g.Wait() // 20个goroutine以及遍历文件的goroutine都执行完
		close(c) 
	}()

	m := make(map[string][md5.Size]byte)
	for r := range c {
		m[r.path] = r.sum
	}
	// Check whether any of the goroutines failed. Since g is accumulating the
	// errors, we don't need to send them (or check for them) in the individual
	// results sent on the channel.
    // 再次调用Wait，依然可以得到group的error信息
	if err := g.Wait(); err != nil {
		return nil, err
	}
	return m, nil
}

```
通过这个例子，你可以学习到多阶段 pipeline 的实现（这个例子是遍历文件夹和计算 md5 两个阶段），还可以学习到如何控制执行子任务的 goroutine 数量。

### 2.3 bilibili ErrGroup 扩展
如果我们无限制地直接调用 ErrGroup 的 Go 方法，就可能会创建出非常多的 goroutine，太多的 goroutine 会带来调度和 GC 的压力，就像[go#34457](https://github.com/golang/go/issues/34457)指出的那样，当前 Go 运行时创建的 g 对象只会增长和重用，不会回收，所以在高并发的情况下，也要尽可能减少 goroutine 的使用。

常用的一个手段就是使用 worker pool(goroutine pool)，或者是类似[containerd/stargz-snapshotter](https://github.com/containerd/stargz-snapshotter/pull/157)的方案，使用前面我们讲的**信号量**，信号量的资源的数量就是可以并行的 goroutine 的数量。

bilibili 实现了一个扩展的 ErrGroup [bilibili/errgroup](https://godoc.org/github.com/bilibili/kratos/pkg/sync/errgroup)，可以使用一个固定数量的 goroutine 处理子任务。如果不设置 goroutine 的数量，那么每个子任务都会比较“放肆地”创建一个 goroutine 并发执行。

除了可以控制并发 goroutine 的数量，它还提供了 2 个功能：
1. cancel，失败的子任务可以 cancel 所有正在执行任务；
2. recover，而且会把 panic 的堆栈信息放到 error 中，避免子任务 panic 导致的程序崩溃。

是，有一点不太好的地方就是，一旦你设置了并发数，超过并发数的子任务需要等到调用者调用 Wait 之后才会执行，而不是只要 goroutine 空闲下来，就去执行。如果不注意这一点的话，可能会出现子任务不能及时处理的情况，这是这个库可以优化的一点。


另外，这个库其实是有一个并发问题的。在高并发的情况下，如果任务数大于设定的 goroutine 的数量，并且这些任务被集中加入到 Group 中，这个库的处理方式是把子任务加入到一个数组中，但是，这个数组不是线程安全的，有并发问题，问题就在于，下面图片中的标记为 96 行的那一行，这一行对 slice 的 append 操作不是线程安全的：

![bilibili_errgroup](/images/go/sync/bilibili_errgroup.png)

我们可以写一个简单的测试程序，运行这个程序的话，你就会发现死锁问题

```go

package main

import (
    "context"
    "fmt"
    "sync/atomic"
    "time"

    "github.com/bilibili/kratos/pkg/sync/errgroup"
)

func main() {
    var g errgroup.Group
    g.GOMAXPROCS(1) // 只使用一个goroutine处理子任务

    var count int64
    g.Go(func(ctx context.Context) error {
        time.Sleep(time.Second) //睡眠5秒，把这个goroutine占住
        return nil
    })

    total := 10000

    for i := 0; i < total; i++ { // 并发一万个goroutine执行子任务，理论上这些子任务都会加入到Group的待处理列表中
        go func() {
            g.Go(func(ctx context.Context) error {
                atomic.AddInt64(&count, 1)
                return nil
            })
        }()
    }

    // 等待所有的子任务完成。理论上10001个子任务都会被完成
    if err := g.Wait(); err != nil {
        panic(err)
    }

    got := atomic.LoadInt64(&count)
    if got != int64(total) {
        panic(fmt.Sprintf("expect %d but got %d", total, got))
    }
}
```

[bilibili errgroup 源码如下](https://github.com/ssoor/open-bilibili/blob/master/library/sync/errgroup.v2/errgroup.go)

```go
// Copyright 2016 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package errgroup provides synchronization, error propagation, and Context
// cancelation for groups of goroutines working on subtasks of a common task.
package errgroup

import (
	"context"
	"fmt"
	"runtime"
	"sync"
)

// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid and does not cancel on error.
type Group struct {
	err     error
	wg      sync.WaitGroup
	errOnce sync.Once

	workerOnce sync.Once
	ch         chan func(ctx context.Context) error
	chs        []func(ctx context.Context) error

	ctx    context.Context
	cancel func()
}

// WithContext create a Group.
// given function from Go will receive this context,
func WithContext(ctx context.Context) *Group {
	return &Group{ctx: ctx}
}

// WithCancel create a new Group and an associated Context derived from ctx.
//
// given function from Go will receive context derived from this ctx,
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
func WithCancel(ctx context.Context) *Group {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{ctx: ctx, cancel: cancel}
}

func (g *Group) do(f func(ctx context.Context) error) {
	ctx := g.ctx
	if ctx == nil {
		ctx = context.Background()
	}
	var err error
	defer func() {
		if r := recover(); r != nil {
			buf := make([]byte, 64<<10)
			buf = buf[:runtime.Stack(buf, false)]
			err = fmt.Errorf("errgroup: panic recovered: %s\n%s", r, buf)
		}
		if err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
		g.wg.Done()
	}()
	err = f(ctx)
}

// GOMAXPROCS set max goroutine to work.
func (g *Group) GOMAXPROCS(n int) {
	if n <= 0 {
		panic("errgroup: GOMAXPROCS must great than 0")
	}
	g.workerOnce.Do(func() {
		g.ch = make(chan func(context.Context) error, n)
		for i := 0; i < n; i++ {
			go func() {
				for f := range g.ch {
					g.do(f)
				}
			}()
		}
	})
}

// Go calls the given function in a new goroutine.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
func (g *Group) Go(f func(ctx context.Context) error) {
	g.wg.Add(1)
	if g.ch != nil {
		select {
		case g.ch <- f:
		default:
			g.chs = append(g.chs, f)
		}
		return
	}
	go g.do(f)
}

// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
	if g.ch != nil {
		for _, f := range g.chs {
			g.ch <- f
		}
	}
	g.wg.Wait()
	if g.ch != nil {
		close(g.ch) // let all receiver exit
	}
	if g.cancel != nil {
		g.cancel()
	}
	return g.err
}
```

### 2.4 neilotoole/errgroup 扩展 
[neilotoole/errgroup](https://github.com/neilotoole/errgroup) 是今年年中新出现的一个 ErrGroup 扩展库，它可以直接替换官方的 ErrGroup，方法都一样，原有功能也一样，只不过增加了可以控制并发 goroutine 的功能。它的方法集如下：

```go
type Group
  func WithContext(ctx context.Context) (*Group, context.Context)
  func WithContextN(ctx context.Context, numG, qSize int) (*Group, context.Context)
  func (g *Group) Go(f func() error)
  func (g *Group) Wait() error
```

新增加的方法 WithContextN，可以**设置并发的 goroutine 数**，以及**等待处理的子任务队列的大小**。当队列满的时候，如果调用 Go 方法，就会被阻塞，直到子任务可以放入到队列中才返回。如果你传给这两个参数的值不是正整数，它就会使用 runtime.NumCPU 代替你传入的参数。

[neilotoole/errgroup](https://github.com/neilotoole/errgroup) 源码如下:

```go
// Package neilotoole/errgroup is an extension of the sync/errgroup
// concept, and much of the code herein is descended from
// or directly copied from that sync/errgroup code which
// has this header comment:
//
// Copyright 2016 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package errgroup is a drop-in alternative to sync/errgroup but
// limited to N goroutines. In effect, neilotoole/errgroup is
// sync/errgroup but with a worker pool of N goroutines.
package errgroup

import (
	"context"
	"runtime"
	"sync"

	"sync/atomic"
)

// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid and does not cancel on error.
//
// This Group implementation differs from sync/errgroup in that instead
// of each call to Go spawning a new Go routine, the f passed to Go
// is sent to a queue channel (qCh), and is picked up by one of N
// worker goroutines. The number of goroutines (numG) and the queue
// channel size (qSize) are args to WithContextN. The zero Group and
// the Group returned by WithContext both use default values (the value
// of runtime.NumCPU) for the numG and qSize args. A side-effect of this
// implementation is that the Go method will block while qCh is full: in
// contrast, errgroup.Group's Go method never blocks (it always spawns
// a new goroutine).
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once
	err     error

	// numG is the maximum number of goroutines that can be started.
	numG int

	// qSize is the capacity of qCh, used for buffering funcs
	// passed to method Go.
	qSize int

	// qCh is the buffer used to hold funcs passed to method Go
	// before they are picked up by worker goroutines.
	qCh chan func() error

	// qMu protects qCh.
	qMu sync.Mutex

	// gCount tracks the number of worker goroutines.
	gCount int64
}

// WithContext returns a new Group and an associated Context derived from ctx.
// It is equivalent to WithContextN(ctx, 0, 0).
func WithContext(ctx context.Context) (*Group, context.Context) {
	return WithContextN(ctx, 0, 0) // zero indicates default values
}

// WithContextN returns a new Group and an associated Context derived from ctx.
//
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
//
// Param numG controls the number of worker goroutines. Param qSize
// controls the size of the queue channel that holds functions passed
// to method Go: while the queue channel is full, Go blocks.
// If numG <= 0, the value of runtime.NumCPU is used; if qSize is
// also <= 0, a qSize of runtime.NumCPU is used.
func WithContextN(ctx context.Context, numG, qSize int) (*Group, context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{cancel: cancel, numG: numG, qSize: qSize}, ctx
}

// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
	g.qMu.Lock()

	if g.qCh != nil {
		// qCh is typically initialized by the first call to method Go.
		// qCh can be nil if Wait is invoked before the first
		// call to Go, hence this check before we close qCh.
		close(g.qCh)
	}

	// Wait for the worker goroutines to finish.
	g.wg.Wait()

	// All of the worker goroutines have finished,
	// so it's safe to set qCh to nil.
	g.qCh = nil

	g.qMu.Unlock()

	if g.cancel != nil {
		g.cancel()
	}

	return g.err
}

// Go adds the given function to a queue of functions that are called
// by one of g's worker goroutines.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
//
// Go may block while g's qCh is full.
func (g *Group) Go(f func() error) {
	g.qMu.Lock()
	if g.qCh == nil {
		// We need to initialize g.

		// The zero value of numG would mean no worker goroutine
		// would be created, which would be daft.
		// We want the "effective" zero value to be runtime.NumCPU.
		if g.numG == 0 {
			// Benchmarking has shown that the optimal numG and
			// qSize values depend on the particular workload. In
			// the absence of any other deciding factor, we somewhat
			// arbitrarily default to NumCPU, which seems to perform
			// reasonably in benchmarks. Users that care about performance
			// tuning will use the WithContextN func to specify the numG
			// and qSize args.
			g.numG = runtime.NumCPU()
			if g.qSize == 0 {
				g.qSize = g.numG
			}
		}

		g.qCh = make(chan func() error, g.qSize)

		// Being that g.Go has been invoked, we'll need at
		// least one goroutine.
		atomic.StoreInt64(&g.gCount, 1)
		g.startG()

		g.qMu.Unlock()

		g.qCh <- f

		return
	}

	g.qCh <- f

	// Check if we can or should start a new goroutine?
	g.maybeStartG()

	g.qMu.Unlock()

}

// maybeStartG might start a new worker goroutine, if
// needed and allowed.
func (g *Group) maybeStartG() {
	if len(g.qCh) == 0 {
		// No point starting a new goroutine if there's
		// nothing in qCh
		return
	}

	// We have at least one item in qCh. Maybe it's time to start
	// a new worker goroutine?
	if atomic.AddInt64(&g.gCount, 1) > int64(g.numG) {
		// Nope: not allowed. Starting a new goroutine would put us
		// over the numG limit, so we back out.
		atomic.AddInt64(&g.gCount, -1)
		return
	}

	// It's safe to start a new worker goroutine.
	g.startG()
}

// startG starts a new worker goroutine.
func (g *Group) startG() {
	g.wg.Add(1)
	go func() {
		defer g.wg.Done()
		defer atomic.AddInt64(&g.gCount, -1)

		var f func() error

		for {
			// Block until f is received from qCh or
			// the channel is closed.
			f = <-g.qCh
			if f == nil {
				// qCh was closed, time for this goroutine
				// to die.
				return
			}

			if err := f(); err != nil {
				g.errOnce.Do(func() {
					g.err = err
					if g.cancel != nil {
						g.cancel()
					}
				})

				return
			}
		}
	}()
}
```

### 2.5 facebookgo/errgroup
[facebookgo/errgroup](https://github.com/facebookarchive/errgroup) Facebook 提供的这个 ErrGroup，其实并不是对 Go 扩展库 ErrGroup 的扩展，而是对标准库 WaitGroup 的扩展。

标准库的 WaitGroup 只提供了 Add、Done、Wait 方法，而且 Wait 方法也没有返回子 goroutine 的 error。而 Facebook 提供的 ErrGroup 提供的 Wait 方法**可以返回 error，而且可以包含多个 error**。子任务在调用 Done 之前，可以把自己的 error 信息设置给 ErrGroup。接着，Wait 在返回的时候，就会把这些 error 信息返回给调用者。

```go
type Group
  func (g *Group) Add(delta int)
  func (g *Group) Done()
  func (g *Group) Error(e error) // 设置 error 给 ErrorGroup，Wait 返回时会返回这些 error
  func (g *Group) Wait() error
```

[facebookgo/errgroup](https://github.com/facebookarchive/errgroup) 源码如下:

```go
// Package errgroup provides a Group that is capable of collecting errors
// as it waits for a collection of goroutines to finish.
package errgroup

import (
	"bytes"
	"sync"
)

// MultiError allows returning a group of errors as one error.
type MultiError []error

// Error returns a concatenated string of all contained errors.
func (m MultiError) Error() string {
	l := len(m)
	if l == 0 {
		panic("MultiError with no errors")
	}
	if l == 1 {
		panic("MultiError with only 1 error")
	}
	var b bytes.Buffer
	b.WriteString("multiple errors: ")
	for i, e := range m {
		b.WriteString(e.Error())
		if i != l-1 {
			b.WriteString(" | ")
		}
	}
	return b.String()
}

// NewMultiError returns nil if all input errors passed in are nil. Otherwise,
// it coalesces all input errors into a single error instance. Useful for
// code like this:
//
//   func doThisAndThat() error {
//     err1 := tryThis()
//     err2 := tryThat()
//     return errgroup.NewMultiError(err1, err2)
//   }
//
func NewMultiError(errs ...error) error {
	var multiErr MultiError
	for _, err := range errs {
		if err != nil {
			multiErr = append(multiErr, err)
		}
	}

	if len(multiErr) == 1 {
		return multiErr[0]
	} else if len(multiErr) > 1 {
		return multiErr
	}
	return nil
}

// Group is similar to a sync.WaitGroup, but allows for collecting errors.
// The collected errors are never reset, so unlike a sync.WaitGroup, this Group
// can only be used _once_. That is, you may only call Wait on it once.
type Group struct {
	wg     sync.WaitGroup
	mu     sync.Mutex
	errors MultiError
}

// Add adds delta, which may be negative. See sync.WaitGroup.Add documentation
// for details.
func (g *Group) Add(delta int) {
	g.wg.Add(delta)
}

// Done decrements the Group counter.
func (g *Group) Done() {
	g.wg.Done()
}

// Error adds an error to return in Wait. The error must not be nil.
func (g *Group) Error(e error) {
	if e == nil {
		panic("error must not be nil")
	}
	g.mu.Lock()
	defer g.mu.Unlock()
	g.errors = append(g.errors, e)
}

// Wait blocks until the Group counter is zero. If no errors were recorded, it
// returns nil. If one error was recorded, it returns it as is. If more than
// one error was recorded it returns a MultiError which is a slice of errors.
func (g *Group) Wait() error {
	g.wg.Wait()
	g.mu.Lock()
	defer g.mu.Unlock()
	errors := g.errors
	l := len(errors)
	if l == 0 {
		return nil
	}
	if l == 1 {
		return errors[0]
	}
	return errors
}
```


下面这些并发原语都是控制一组子 goroutine 执行的面向特定场景的并发原语，当你遇见这些特定场景时，就可以参考这些库。

## 3. go-pkgz/syncs
[go-pkgz/syncs](https://github.com/go-pkgz/syncs) 提供了两个 Group 并发原语，分别是 SizedGroup 和 ErrSizedGroup:

### 3.1 SizedGroup
SizedGroup 内部是使用信号量和 WaitGroup 实现的，它通过信号量控制并发的 goroutine 数量，或者是不控制 goroutine 数量，只控制子任务并发执行时候的数量（通过）。

默认情况下，SizedGroup 控制的是子任务的并发数量，而不是 goroutine 的数量。在这种方式下，每次调用 Go 方法都不会被阻塞，而是新建一个 goroutine 去执行。如果想控制 goroutine 的数量，你可以使用 syncs.Preemptive 设置这个并发原语的可选项。如果设置了这个可选项，但在调用 Go 方法的时候没有可用的 goroutine，那么调用者就会等待，直到有 goroutine 可以处理这个子任务才返回，这个控制在内部是使用信号量实现的。

#### SizedGroup 使用示例
```go

package main

import (
    "context"
    "fmt"
    "sync/atomic"
    "time"

    "github.com/go-pkgz/syncs"
)

func main() {
    // 设置goroutine数是10
    swg := syncs.NewSizedGroup(10)
    // swg := syncs.NewSizedGroup(10, syncs.Preemptive)
    var c uint32

    // 执行1000个子任务，只会有10个goroutine去执行
    for i := 0; i < 1000; i++ {
        swg.Go(func(ctx context.Context) {
            time.Sleep(5 * time.Millisecond)
            atomic.AddUint32(&c, 1)
        })
    }

    // 等待任务完成
    swg.Wait()
    // 输出结果
    fmt.Println(c)
}
```

#### SizedGroup 实现

```go
package syncs

import (
	"context"
	"sync"
)

// SizedGroup has the same role as WaitingGroup but adds a limit of the amount of goroutines started concurrently.
// Uses similar Go() scheduling as errgrp.Group, thread safe.
// SizedGroup interface enforces constructor usage and doesn't allow direct creation of sizedGroup
type SizedGroup struct {
	options
	wg   sync.WaitGroup
	sema sync.Locker
}

// NewSizedGroup makes wait group with limited size alive goroutines
func NewSizedGroup(size int, opts ...GroupOption) *SizedGroup {
	res := SizedGroup{sema: NewSemaphore(size)}
	res.options.ctx = context.Background()
	for _, opt := range opts {
		opt(&res.options)
	}
	return &res
}

// Go calls the given function in a new goroutine.
// Every call will be unblocked, but some goroutines may wait if semaphore locked.
func (g *SizedGroup) Go(fn func(ctx context.Context)) {

	canceled := func() bool {
		select {
		case <-g.ctx.Done():
			return true
		default:
			return false
		}
	}

	if canceled() {
		return
	}

	g.wg.Add(1)

	if g.preLock {
		g.sema.Lock()
	}

	go func() {
		defer g.wg.Done()

		if canceled() {
			return
		}

		if !g.preLock {
			g.sema.Lock()
		}

		fn(g.ctx)
		g.sema.Unlock()
	}()
}

// Wait blocks until the SizedGroup counter is zero.
// See sync.WaitGroup documentation for more information.
func (g *SizedGroup) Wait() {
	g.wg.Wait()
}
```

SizedGroup 使用到的信号量实现:

```go
package syncs

import "sync"

// Semaphore implementation, counted lock only. Implements sync.Locker interface, thread safe.
type semaphore struct {
	sync.Locker
	ch chan struct{}
}

// NewSemaphore makes Semaphore with given capacity
func NewSemaphore(capacity int) sync.Locker {
	if capacity <= 0 {
		capacity = 1
	}
	return &semaphore{ch: make(chan struct{}, capacity)}
}

// Lock acquires semaphore, can block if out of capacity.
func (s *semaphore) Lock() {
	s.ch <- struct{}{}
}

// Unlock releases semaphore, can block if nothing acquired before.
func (s *semaphore) Unlock() {
	<-s.ch
}
```

### 3.2 ErrSizedGroup

ErrSizedGroup 为 SizedGroup 提供了 error 处理的功能，它的功能和 Go 官方扩展库的功能一样，就是等待子任务完成并返回第一个出现的 error。不过，它还提供了额外的功能
1. 可以控制并发的 goroutine 数量，这和 SizedGroup 的功能一样
2. 如果设置了 termOnError，子任务出现第一个错误的时候会 cancel Context，而且后续的 Go 调用会直接返回，Wait 调用者会得到这个错误，这相当于是遇到错误快速返回。如果没有设置 termOnError，Wait 会返回所有的子任务的错误

不过，ErrSizedGroup 和 SizedGroup 设计得不太一致的地方是，SizedGroup 可以把 Context 传递给子任务，这样可以通过 cancel 让子任务中断执行，但是 ErrSizedGroup 却没有实现。ErrsizedGroup 的实现类似，但是增加了收集 Error 的能力:

#### ErrSizedGroup 实现
```go
package syncs

import (
	"fmt"
	"strings"
	"sync"
)

// ErrSizedGroup is a SizedGroup with error control. Works the same as errgrp.Group, i.e. returns first error.
// Can work as regular errgrp.Group or with early termination. Thread safe.
// ErrSizedGroup interface enforces constructor usage and doesn't allow direct creation of errSizedGroup
type ErrSizedGroup struct {
	options
	wg   sync.WaitGroup
	sema sync.Locker

	err     *multierror
	errLock sync.RWMutex
	errOnce sync.Once
}

// NewErrSizedGroup makes wait group with limited size alive goroutines.
// By default all goroutines will be started but will wait inside. For limited number of goroutines use Preemptive() options.
// TermOnErr will skip (won't start) all other goroutines if any error returned.
func NewErrSizedGroup(size int, options ...GroupOption) *ErrSizedGroup {

	res := ErrSizedGroup{
		sema: NewSemaphore(size),
		err:  new(multierror),
	}

	for _, opt := range options {
		opt(&res.options)
	}

	return &res
}

// Go calls the given function in a new goroutine.
// The first call to return a non-nil error cancels the group if termOnError; its error will be
// returned by Wait. If no termOnError all errors will be collected in multierror.
func (g *ErrSizedGroup) Go(f func() error) {

	g.wg.Add(1)

	if g.preLock {
		g.sema.Lock()
	}

	go func() {
		defer g.wg.Done()

		// terminated will be true if any error happened before and g.termOnError
		terminated := func() bool {
			if !g.termOnError {
				return false
			}
			g.errLock.RLock()
			defer g.errLock.RUnlock()
			return g.err.errorOrNil() != nil
		}

		if terminated() {
			return // terminated due prev error, don't run anything in this group anymore
		}

		if !g.preLock {
			g.sema.Lock()
		}

		if err := f(); err != nil {

			g.errLock.Lock()
			g.err = g.err.append(err)
			g.errLock.Unlock()

			g.errOnce.Do(func() { // call context cancel once
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
		g.sema.Unlock()
	}()
}

// Wait blocks until all function calls from the Go method have returned, then
// returns all errors (if any) wrapped with multierror from them.
func (g *ErrSizedGroup) Wait() error {
	g.wg.Wait()
	if g.cancel != nil {
		g.cancel()
	}
	return g.err.errorOrNil()
}

type multierror struct {
	errors []error
	lock   sync.Mutex
}

func (m *multierror) append(err error) *multierror {
	m.lock.Lock()
	m.errors = append(m.errors, err)
	m.lock.Unlock()
	return m
}

func (m *multierror) errorOrNil() error {
	m.lock.Lock()
	defer m.lock.Unlock()
	if len(m.errors) == 0 {
		return nil
	}
	return m
}

// Error returns multierror string
func (m *multierror) Error() string {
	m.lock.Lock()
	defer m.lock.Unlock()
	if len(m.errors) == 0 {
		return ""
	}

	errs := []string{}

	for n, e := range m.errors {
		errs = append(errs, fmt.Sprintf("[%d] {%s}", n, e.Error()))
	}
	return fmt.Sprintf("%d error(s) occurred: %s", len(m.errors), strings.Join(errs, ", "))
}
```

## 4. gollback
[gollback](https://github.com/vardius/gollback)也是用来处理一组子任务的执行的，不过它解决了 ErrGroup 收集子任务返回结果的痛点。它的方法会把结果和 error 信息都返回。

gollback 提供了如下三个方法: All, Race, Retry

### 4.1 All
All:
- 签名: `func All(ctx context.Context, fns ...AsyncFunc) ([]interface{}, []error)`
- 执行: 它会等待所有的异步函数（AsyncFunc）都执行完才返回，而且返回结果的顺序和传入的函数的顺序保持一致。
- 返回: 第一个返回参数是子任务的执行结果，第二个参数是子任务执行时的错误信息
- 异步函数: 
    - `type AsyncFunc func(ctx context.Context) (interface{}, error)`
    - ctx 会被传递给子任务。如果你 cancel 这个 ctx，可以取消子任务

```go

package main

import (
  "context"
  "errors"
  "fmt"
  "github.com/vardius/gollback"
  "time"
)

func main() {
  rs, errs := gollback.All( // 调用All方法
    context.Background(),
    func(ctx context.Context) (interface{}, error) { 
      time.Sleep(3 * time.Second)
      return 1, nil // 第一个任务没有错误，返回1
    },
    func(ctx context.Context) (interface{}, error) {
      return nil, errors.New("failed") // 第二个任务返回一个错误
    },
    func(ctx context.Context) (interface{}, error) {
      return 3, nil // 第三个任务没有错误，返回3
    },
  )

  fmt.Println(rs) // 输出子任务的结果
  fmt.Println(errs) // 输出子任务的错误信息
}
```

### 4.2 Race
Race:
- 签名: `func Race(ctx context.Context, fns ...AsyncFunc) (interface{}, error)`
- 返回: 跟 All 方法类似，只不过，在使用 Race 方法的时候，只要一个异步函数执行没有错误，就立马返回，而不会返回所有的子任务信息。如果所有的子任务都没有成功，就会返回最后一个 error 信息
- 注意: 如果有一个正常的子任务的结果返回，Race 会把传入到其它子任务的 Context cancel 掉，这样子任务就可以中断自己的执行

### 4.3 Retry
Retry:
- 签名: `func Retry(ctx context.Context, retires int, fn AsyncFunc) (interface{}, error)`
- 作用: Retry 不是执行一组子任务，而是执行一个子任务
- 返回:
    - 如果子任务执行失败，它会尝试一定的次数，
    - 如果一直不成功 ，就会返回失败错误  
    - 如果执行成功，它会立即返回
    - 如果 retires 等于 0，它会永远尝试，直到成功

```go

package main

import (
  "context"
  "errors"
  "fmt"
  "github.com/vardius/gollback"
  "time"
)

func main() {
  ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()

  // 尝试5次，或者超时返回
  res, err := gollback.Retry(ctx, 5, func(ctx context.Context) (interface{}, error) {
    return nil, errors.New("failed")
  })

  fmt.Println(res) // 输出结果
  fmt.Println(err) // 输出错误信息
} 
```

### 4.4 gollback 实现
```go
package gollback

import (
	"context"
	"errors"
	"sync"
)

var ErrNoCallbacks = errors.New("no callback to run")

// AsyncFunc represents asynchronous function
type AsyncFunc func(ctx context.Context) (interface{}, error)

type response struct {
	res interface{}
	err error
}

// Race method returns a response as soon as one of the callbacks executes without an error,
// otherwise last error is returned
// will panic if context is nil
func Race(ctx context.Context, fns ...AsyncFunc) (interface{}, error) {
	if ctx == nil {
		panic("nil context provided")
	}
	if len(fns) == 0 {
		return nil, ErrNoCallbacks
	}

	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	out := make(chan *response, 1)
	defer close(out)
	responses := make(chan *response, len(fns))
	defer close(responses)

	var responded bool
	var lock sync.Mutex
	var errCount int
	go func() {
		for {
			select {
			case <-ctx.Done():
				lock.Lock()
				if !responded {
					responded = true
					out <- &response{
						err: ctx.Err(),
					}
					lock.Unlock()
					return
				}
				lock.Unlock()
			case r, more := <-responses:
				lock.Lock()
				if !more || (!responded && r.err == nil) || (errCount == len(fns)) {
					responded = true
					out <- r
					lock.Unlock()
					return
				}
				lock.Unlock()
			}
		}
	}()

	for _, fn := range fns {
		go func(f AsyncFunc) {
			var r response
			r.res, r.err = f(ctx)

			lock.Lock()
			defer lock.Unlock()
			if r.err != nil {
				errCount ++
			}
			if !responded {
				responses <- &r
			}
		}(fn)
	}

	r := <-out

	return r.res, r.err
}

// All method returns when all the callbacks passed as an iterable have finished,
// returned responses and errors are ordered according to callback order
// will panic if context is nil
func All(ctx context.Context, fns ...AsyncFunc) ([]interface{}, []error) {
	if ctx == nil {
		panic("nil context provided")
	}

	rs := make([]interface{}, len(fns))
	errs := make([]error, len(fns))

	var wg sync.WaitGroup
	wg.Add(len(fns))

	for i, fn := range fns {
		go func(index int, f AsyncFunc) {
			defer wg.Done()

			var r response
			r.res, r.err = f(ctx)

			rs[index] = r.res
			errs[index] = r.err
		}(i, fn)
	}

	wg.Wait()

	return rs, errs
}

// Retry method retries callback given amount of times until it executes without an error,
// when retries = 0 it will retry infinitely
// will panic if context is nil
func Retry(ctx context.Context, retires int, fn AsyncFunc) (interface{}, error) {
	if ctx == nil {
		panic("nil context provided")
	}

	i := 1

	for {
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		default:
			var r response
			r.res, r.err = fn(ctx)

			if r.err == nil || i == retires {
				return r.res, r.err
			}

			i++
		}
	}
}
```

## 5. Hunch
[Hunch](https://github.com/AaronJan/Hunch) 提供的功能和 gollback 类似，不过它提供的方法更多包括:
1. All
2. Take
3. Last
4. Retry
5. Waterfall

它定义了执行子任务的函数，这和 gollback 的 AyncFunc 是一样的：

```go
type Executable func(context.Context) (interface{}, error)
```

### 5.1 All
All:
- 签名: `func All(parentCtx context.Context, execs ...Executable) ([]interface{}, error)`
- 作用: 传入一组可执行的函数（子任务），返回子任务的执行结果
- 区别: 和 gollback 的 All 方法不一样的是，一旦一个子任务出现错误，它就会返回错误信息，执行结果（第一个返回参数）为 nil。

### 5.2 Take
Take:
- 签名: `func Take(parentCtx context.Context, num int, execs ...Executable) ([]interface{}, error)`
- 作用: 
    - 可以指定 num 参数，只要有 num 个子任务正常执行完没有错误，这个方法就会返回这几个子任务的结果
    - 一旦一个子任务出现错误，它就会返回错误信息，执行结果（第一个返回参数）为 nil。

### 5.3 Last
Last:
- 签名: `func Last(parentCtx context.Context, num int, execs ...Executable) ([]interface{}, error)`
- 作用: 
    - 只返回最后 num 个正常执行的、没有错误的子任务的结果
    - 一旦一个子任务出现错误，它就会返回错误信息，执行结果（第一个返回参数）为 nil

### 5.4 Retry
Retry:
- 签名: `func Retry(parentCtx context.Context, retries int, fn Executable) (interface{}, error)`
- 作用: 
    - 它的功能和 gollback 的 Retry 方法的功能一样，如果子任务执行出错，就会不断尝试，直到成功或者是达到重试上限。
    - 如果达到重试上限，就会返回错误
    - 如果 retries 等于 0，它会不断尝试

### 5.5 Waterfall
Waterfall:
- 签名: `func Waterfall(parentCtx context.Context, execs ...ExecutableInSequence) (interface{}, error)`
- 作用: 
    - 它其实是一个 pipeline 的处理方式，所有的子任务都是串行执行的，
    - 前一个子任务的执行结果会被当作参数传给下一个子任务，直到所有的任务都完成，返回最后的执行结果
    - 一旦一个子任务出现错误，它就会返回错误信息，执行结果（第一个返回参数）为 nil

[Hunch](https://github.com/AaronJan/Hunch) 的源码如下:

```go
// Package hunch provides functions like: `All`, `First`, `Retry`, `Waterfall` etc., that makes asynchronous flow control more intuitive.
package hunch

import (
	"context"
	"fmt"
	"sort"
	"sync"
)

// Executable represents a singular logic block.
// It can be used with several functions.
type Executable func(context.Context) (interface{}, error)

// ExecutableInSequence represents one of a sequence of logic blocks.
type ExecutableInSequence func(context.Context, interface{}) (interface{}, error)

// IndexedValue stores the output of Executables,
// along with the index of the source Executable for ordering.
type IndexedValue struct {
	Index int
	Value interface{}
}

// IndexedExecutableOutput stores both output and error values from a Excetable.
type IndexedExecutableOutput struct {
	Value IndexedValue
	Err   error
}

func pluckVals(iVals []IndexedValue) []interface{} {
	vals := []interface{}{}
	for _, val := range iVals {
		vals = append(vals, val.Value)
	}

	return vals
}

func sortIdxVals(iVals []IndexedValue) []IndexedValue {
	sorted := make([]IndexedValue, len(iVals))
	copy(sorted, iVals)
	sort.SliceStable(
		sorted,
		func(i, j int) bool {
			return sorted[i].Index < sorted[j].Index
		},
	)

	return sorted
}

// Take returns the first `num` values outputted by the Executables.
func Take(parentCtx context.Context, num int, execs ...Executable) ([]interface{}, error) {
	execCount := len(execs)

	if num > execCount {
		num = execCount
	}

	// Create a new sub-context for possible cancelation.
	ctx, cancel := context.WithCancel(parentCtx)
	defer cancel()

	output := make(chan IndexedExecutableOutput, 1)
	go runExecs(ctx, output, execs)

	fail := make(chan error, 1)
	success := make(chan []IndexedValue, 1)
	go takeUntilEnough(fail, success, min(len(execs), num), output)

	select {

	case <-parentCtx.Done():
		// Stub comment to fix a test coverage bug.
		return nil, parentCtx.Err()

	case err := <-fail:
		cancel()
		if parentCtxErr := parentCtx.Err(); parentCtxErr != nil {
			return nil, parentCtxErr
		}
		return nil, err

	case uVals := <-success:
		cancel()
		return pluckVals(uVals), nil
	}
}

func runExecs(ctx context.Context, output chan<- IndexedExecutableOutput, execs []Executable) {
	var wg sync.WaitGroup
	for i, exec := range execs {
		wg.Add(1)

		go func(i int, exec Executable) {
			val, err := exec(ctx)
			if err != nil {
				output <- IndexedExecutableOutput{
					IndexedValue{i, nil},
					err,
				}
				wg.Done()
				return
			}

			output <- IndexedExecutableOutput{
				IndexedValue{i, val},
				nil,
			}
			wg.Done()
		}(i, exec)
	}

	wg.Wait()
	close(output)
}

func takeUntilEnough(fail chan error, success chan []IndexedValue, num int, output chan IndexedExecutableOutput) {
	uVals := make([]IndexedValue, num)

	enough := false
	outputCount := 0
	for r := range output {
		if enough {
			continue
		}

		if r.Err != nil {
			enough = true
			fail <- r.Err
			continue
		}

		uVals[outputCount] = r.Value
		outputCount++

		if outputCount == num {
			enough = true
			success <- uVals
			continue
		}
	}
}

// All returns all the outputs from all Executables, order guaranteed.
func All(parentCtx context.Context, execs ...Executable) ([]interface{}, error) {
	// Create a new sub-context for possible cancelation.
	ctx, cancel := context.WithCancel(parentCtx)
	defer cancel()

	output := make(chan IndexedExecutableOutput, 1)
	go runExecs(ctx, output, execs)

	fail := make(chan error, 1)
	success := make(chan []IndexedValue, 1)
	go takeUntilEnough(fail, success, len(execs), output)

	select {

	case <-parentCtx.Done():
		// Stub comment to fix a test coverage bug.
		return nil, parentCtx.Err()

	case err := <-fail:
		cancel()
		if parentCtxErr := parentCtx.Err(); parentCtxErr != nil {
			return nil, parentCtxErr
		}
		return nil, err

	case uVals := <-success:
		cancel()
		return pluckVals(sortIdxVals(uVals)), nil
	}
}

/*
Last returns the last `num` values outputted by the Executables.
*/
func Last(parentCtx context.Context, num int, execs ...Executable) ([]interface{}, error) {
	execCount := len(execs)
	if num > execCount {
		num = execCount
	}
	start := execCount - num

	vals, err := Take(parentCtx, execCount, execs...)
	if err != nil {
		return nil, err
	}

	return vals[start:], err
}

// MaxRetriesExceededError stores how many times did an Execution run before exceeding the limit.
// The retries field holds the value.
type MaxRetriesExceededError struct {
	retries int
}

func (err MaxRetriesExceededError) Error() string {
	var word string
	switch err.retries {
	case 0:
		word = "infinity"
	case 1:
		word = "1 time"
	default:
		word = fmt.Sprintf("%v times", err.retries)
	}

	return fmt.Sprintf("Max retries exceeded (%v).\n", word)
}

// Retry attempts to get a value from an Executable instead of an Error.
// It will keeps re-running the Executable when failed no more than `retries` times.
// Also, when the parent Context canceled, it returns the `Err()` of it immediately.
func Retry(parentCtx context.Context, retries int, fn Executable) (interface{}, error) {
	ctx, cancel := context.WithCancel(parentCtx)
	defer cancel()

	c := 0
	fail := make(chan error, 1)
	success := make(chan interface{}, 1)

	for {
		go func() {
			val, err := fn(ctx)
			if err != nil {
				fail <- err
				return
			}
			success <- val
		}()

		select {
		//
		case <-parentCtx.Done():
			// Stub comment to fix a test coverage bug.
			return nil, parentCtx.Err()

		case <-fail:
			if parentCtxErr := parentCtx.Err(); parentCtxErr != nil {
				return nil, parentCtxErr
			}

			c++
			if retries == 0 || c < retries {
				continue
			}
			return nil, MaxRetriesExceededError{c}

		case val := <-success:
			return val, nil
		}
	}
}

// Waterfall runs `ExecutableInSequence`s one by one,
// passing previous result to next Executable as input.
// When an error occurred, it stop the process then returns the error.
// When the parent Context canceled, it returns the `Err()` of it immediately.
func Waterfall(parentCtx context.Context, execs ...ExecutableInSequence) (interface{}, error) {
	ctx, cancel := context.WithCancel(parentCtx)
	defer cancel()

	var lastVal interface{}
	execCount := len(execs)
	i := 0
	fail := make(chan error, 1)
	success := make(chan interface{}, 1)

	for {
		go func() {
			val, err := execs[i](ctx, lastVal)
			if err != nil {
				fail <- err
				return
			}
			success <- val
		}()

		select {

		case <-parentCtx.Done():
			// Stub comment to fix a test coverage bug.
			return nil, parentCtx.Err()

		case err := <-fail:
			if parentCtxErr := parentCtx.Err(); parentCtxErr != nil {
				return nil, parentCtxErr
			}

			return nil, err

		case val := <-success:
			lastVal = val
			i++
			if i == execCount {
				return val, nil
			}

			continue
		}
	}
}

func min(x, y int) int {
	if x > y {
		return y
	}
	return x
}
```

### 5.6 总结
gollback 和 Hunch 是属于同一类的并发原语，对一组子任务的执行结果，可以选择一个结果或者多个结果

## 6. schedgroup
[schedgroup](https://github.com/mdlayher/schedgroup) 是一个和时间相关的处理一组 goroutine 的并发原语，是 Matt Layher 开发的 worker pool，可以**指定任务在某个时间或者某个时间之后执行**。他在 GopherCon Europe 2020 大会上专门介绍了这个并发原语：[schedgroup: a timer-based goroutine concurrency primitive](https://talks.godoc.org/github.com/mdlayher/talks/conferences/2020/gopherconeu/schedgroup.slide) 

schedgroup 包含的方法如下：

```go

type Group
  func New(ctx context.Context) *Group
  func (g *Group) Delay(delay time.Duration, fn func())
  func (g *Group) Schedule(when time.Time, fn func())
  func (g *Group) Wait() error
```

1. Delay 和 Schedule: 功能其实是一样的，都是用来指定在某个时间或者之后执行一个函数。
2. Wait: 会阻塞调用者，直到之前安排的所有子任务都执行完才返回。如果 Context 被取消，那么，Wait 方法会返回这个 cancel error
    - 如果调用了 Wait 方法，你就不能再调用它的 Delay 和 Schedule 方法，否则会 panic。
    - Wait 方法只能调用一次，如果多次调用的话，就会 panic

你可能认为，简单地使用 timer 就可以实现这个功能。其实，如果只有几个子任务，使用 timer 不是问题，但**一旦有大量的子任务，而且还要能够 cancel，那么，使用 timer 的话，CPU 资源消耗就比较大了。所以，schedgroup 在实现的时候，就使用 container/heap**，按照子任务的执行时间进行排序，这样可以避免使用大量的 timer，从而提高性能。

```go

sg := schedgroup.New(context.Background())

// 设置子任务分别在100、200、300之后执行
for i := 0; i < 3; i++ {
    n := i + 1
    sg.Delay(time.Duration(n)*100*time.Millisecond, func() {
        log.Println(n) //输出任务编号
    })
}

// 等待所有的子任务都完成
if err := sg.Wait(); err != nil {
    log.Fatalf("failed to wait: %v", err)
}
```

[schedgroup](https://github.com/mdlayher/schedgroup) 源码如下:

```go
package schedgroup

import (
	"container/heap"
	"context"
	"sync"
	"sync/atomic"
	"time"
)

// Although unnecessary, explicit break labels should be used in all select
// statements in this package so that test coverage tools are able to identify
// which cases have been triggered.

// A Group is a goroutine worker pool which schedules tasks to be performed
// after a specified time. A Group must be created with the New constructor.
// Once Wait is called, New must be called to create a new Group to schedule
// more tasks.
type Group struct {
	// Atomics must come first per sync/atomic.
	waiting uint32

	// Context/cancelation support.
	ctx    context.Context
	cancel func()

	// Task runner and a heap of tasks to be run.
	wg    sync.WaitGroup
	mu    sync.Mutex
	tasks tasks

	// Signals for when a task is added and how many tasks remain on the heap.
	addC chan struct{}
	lenC chan int
}

// New creates a new Group which will use ctx for cancelation. If cancelation
// is not a concern, use context.Background().
func New(ctx context.Context) *Group {
	// Monitor goroutine context and cancelation.
	mctx, cancel := context.WithCancel(ctx)

	g := &Group{
		ctx:    ctx,
		cancel: cancel,

		addC: make(chan struct{}),
		lenC: make(chan int),
	}

	g.wg.Add(1)
	go func() {
		defer g.wg.Done()
		g.monitor(mctx)
	}()

	return g
}

// Delay schedules a function to run at or after the specified delay. Delay
// is a convenience wrapper for Schedule which adds delay to the current time.
// Specifying a negative delay will cause the task to be scheduled immediately.
//
// If Delay is called after a call to Wait, Delay will panic.
func (g *Group) Delay(delay time.Duration, fn func()) {
	g.Schedule(time.Now().Add(delay), fn)
}

// Schedule schedules a function to run at or after the specified time.
// Specifying a past time will cause the task to be scheduled immediately.
//
// If Schedule is called after a call to Wait, Schedule will panic.
func (g *Group) Schedule(when time.Time, fn func()) {
	if atomic.LoadUint32(&g.waiting) != 0 {
		panic("schedgroup: attempted to schedule task after Group.Wait was called")
	}

	g.mu.Lock()
	defer g.mu.Unlock()

	heap.Push(&g.tasks, task{
		Deadline: when,
		Call:     fn,
	})

	// Notify monitor that a new task has been pushed on to the heap.
	select {
	case g.addC <- struct{}{}:
		break
	default:
		break
	}
}

// Wait waits for the completion of all scheduled tasks, or for cancelation of
// the context passed to New. Wait will only returns errors due to context
// cancelation. If no context is associated the the Group, wait never returns
// an error.
//
// Once Wait is called, any further calls to Delay or Schedule will panic. If
// Wait is called more than once, Wait will panic.
func (g *Group) Wait() error {
	if v := atomic.SwapUint32(&g.waiting, 1); v != 0 {
		panic("schedgroup: multiple calls to Group.Wait")
	}

	// Context cancelation takes priority.
	if err := g.ctx.Err(); err != nil {
		return err
	}

	// See if the task heap is already empty. If so, we can exit early.
	g.mu.Lock()
	if g.tasks.Len() == 0 {
		// Release the mutex immediately so that any running jobs are able to
		// complete and send on g.lenC.
		g.mu.Unlock()
		g.cancel()
		g.wg.Wait()
		return nil
	}
	g.mu.Unlock()

	// Wait on context cancelation or for the number of items in the heap
	// to reach 0.
	var n int
	for {
		select {
		case <-g.ctx.Done():
			return g.ctx.Err()
		case n = <-g.lenC:
			// Context cancelation takes priority.
			if err := g.ctx.Err(); err != nil {
				return err
			}
		}

		if n == 0 {
			// No more tasks left, cancel the monitor goroutine and wait for
			// all tasks to complete.
			g.cancel()
			g.wg.Wait()
			return nil
		}
	}
}

// monitor triggers tasks at the interval specified by g.Interval until ctx
// is canceled.
func (g *Group) monitor(ctx context.Context) {
	t := time.NewTimer(0)
	defer t.Stop()

	for {
		if ctx.Err() != nil {
			// Context canceled.
			return
		}

		now := time.Now()
		var tickC <-chan time.Time

		// Start any tasks that are ready as of now.
		next := g.trigger(now)
		if !next.IsZero() {
			// Wait until the next scheduled task is ready.
			t.Reset(next.Sub(now))
			tickC = t.C
		} else {
			t.Stop()
		}

		select {
		case <-ctx.Done():
			// Context canceled.
			return
		case <-g.addC:
			// A new task was added, check task heap again.
			//lint:ignore SA4011 intentional break for code coverage
			break
		case <-tickC:
			// An existing task should be ready as of now.
			//lint:ignore SA4011 intentional break for code coverage
			break
		}
	}
}

// trigger checks for scheduled tasks and runs them if they are scheduled
// on or after the time specified by now.
func (g *Group) trigger(now time.Time) time.Time {
	g.mu.Lock()
	defer func() {
		// Notify how many tasks are left on the heap so Wait can stop when
		// appropriate.
		select {
		case g.lenC <- g.tasks.Len():
			break
		default:
			// Wait hasn't been called.
			break
		}

		g.mu.Unlock()
	}()

	for g.tasks.Len() > 0 {
		next := &g.tasks[0]
		if next.Deadline.After(now) {
			// Earliest scheduled task is not ready.
			return next.Deadline
		}

		// This task is ready, pop it from the heap and run it.
		t := heap.Pop(&g.tasks).(task)
		g.wg.Add(1)
		go func() {
			defer g.wg.Done()
			t.Call()
		}()
	}

	return time.Time{}
}

// A task is a function which is called after the specified deadline.
type task struct {
	Deadline time.Time
	Call     func()
}

// tasks implements heap.Interface.
type tasks []task

var _ heap.Interface = &tasks{}

func (pq tasks) Len() int            { return len(pq) }
func (pq tasks) Less(i, j int) bool  { return pq[i].Deadline.Before(pq[j].Deadline) }
func (pq tasks) Swap(i, j int)       { pq[i], pq[j] = pq[j], pq[i] }
func (pq *tasks) Push(x interface{}) { *pq = append(*pq, x.(task)) }
func (pq *tasks) Pop() (item interface{}) {
	n := len(*pq)
	item, *pq = (*pq)[n-1], (*pq)[:n-1]
	return item
}
```

## 7. go-waitgroup
[go-waitgroup](https://github.com/pieterclaerhout/go-waitgroup) 是一个带并发控制的 waitGroup。功能上跟 SizedGroup 差不多，这里不再赘述。

## 8. 个人理解
今天这篇文章，内容很多，看着有些乱，但是核心就是对分组执行的任务的并发控制。可以看成是对 WaitGroup 的扩展。扩展的点主要是针对 WaitGroup 的不足之处:
1. 并发数没有控制，包括: go-waitgroup，SizedGroup，neilotoole/errgroup，Go 官方的 ErrGroup
2. 无法收集子 goroutine 返回的错误，包括: facebookgo/errgroup，ErrSizedGroup

而 Hunch、gollback、schedgroup 则是对分组任务执行语义上的扩展: 等待全部执行完成，等待某一个执行完成，定时执行。


## 参考
本文内容摘录自:
1. [极客专栏-鸟叔的 Go 并发编程实战](https://time.geekbang.org/column/intro/100061801?tab=catalog)
