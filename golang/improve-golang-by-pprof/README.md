使用 proof 分析程序性能
-----------------------------
Go 语言中的 pprof 工具不仅可以通过分析找到程序中的错误（内存泄漏、数据竞争、协程泄漏），也可以为程序性能优化提供参考依据和基准指标。


### 使用 pprof 工具
使用 pprof 工具通常有两个步骤：收集样本和分析样本。

#### 收集样本的方式
收集样本主要有两种方式：

- 一种是引入 `net/http/pprof` 并在程序中启动 HTTP 服务，通过访问 `/debug/pprof` 的方式收集所需的样本
- 另一种是直接在需要收集样本分析的位置嵌入收集方法。

最常见的样本数据包括：堆内存分析（heap）、协程栈分析（goroutine）、CPU 占用分析（profile）、程序运行时事件分析（trace）。

#### 测试用例
先准备一个测试用例用来说明后续的样本分析方式：
```go
package main

import (
	"fmt"
	"io"
	"math/rand"
	"net/http"
	_ "net/http/pprof"
	"sync"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

func goroutines() {
	for {
		var wg sync.WaitGroup
		for i := 0; i < 100; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()

				time.Sleep(time.Millisecond * time.Duration(rand.Int63n(1000)))
			}()
		}

		wg.Wait()
	}
}

func heaps() {
	for {
		var wg sync.WaitGroup
		for i := 0; i < 100; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()

				bs := make([]byte, 1024*rand.Int63n(1024))
				time.Sleep(time.Millisecond * time.Duration(rand.Int63n(1000)))
				_, _ = fmt.Fprintf(io.Discard, "malloc %d bytes\n", len(bs))
			}()
		}

		wg.Wait()
	}
}

func costly() {
	for {
		for i := 0; i < 1_000_000_007; i++ {
		}
		time.Sleep(time.Second)
	}
}

func main() {
	go heaps()
	go costly()
	go goroutines()
	if err := http.ListenAndServe(":8088", nil); err != nil {
		panic(err)
	}
}
```
> 注意：以下所有指标分析都是在以上程序运行的情况下进行的。

#### 在线查看指标数据
在启动以上程序之后，我们可以直接在浏览器中打开 `http://localhost:8088/debug/pprof` 就可以看到所有提供的指标项目了。

### 性能指标分析
在进入 pprof 工具之后，可以使用命令的形式获取想要查看的指标，常用的命令有：

- `top` 会以 `flat` 从大到小排序列出表格。其中 `flat` 表示当前正在统计的值，`cum` 是按照函数调用关系累计的 `flat` 值的和。
   - `top -cum` 会以 `cum` 从大到小排序列出表格。
- `tree` 会以调用链的方式输出，在其中能看到该值所在的上下文、堆栈信息。
- `list` 会直接将指标所在的代码列出并指明该指标的所在位置。
   - **该命令会加载源代码文件进行展示，所以要求在项目根目录执行相应的命令**
- `web` 通过 `graphviz` 可视化的方式在浏览器上查看结果。

其他任何的命令可以通过 `help [command]` 的方式查看帮助。


#### 堆内存分析
我们可以在终端中使用如下命令获取堆内存指标：
```bash
$ go tool pprof http://localhost:8088/debug/pprof/heap
Fetching profile over HTTP from http://localhost:8088/debug/pprof/heap
Type: inuse_space
Time: Feb 11, 2023 at 3:06pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```
从输出的信息可以看到当前所查看指标的类型为 `inuse_space`。我们可以直接键入以下命令来切换所要查看分析的堆指标类型：

- `alloc_objects`：已经分配的**对象**数量
- `alloc_space`：已经分配的**内存**数量
- `inuse_objects`：正在使用的**对象**数量
- `inuse_space`：正在使用的**内存**数量

我们首先通过 `top` 命令按照占有内存从大到小列出所有函数名称：
```bash
(pprof) top
Showing nodes accounting for 21248.53kB, 100% of 21248.53kB total
Showing top 10 nodes out of 16
      flat  flat%   sum%        cum   cum%
18173.87kB 85.53% 85.53% 18173.87kB 85.53%  main.heaps.func1
 2050.25kB  9.65% 95.18%  2050.25kB  9.65%  runtime.allocm
 1024.41kB  4.82%   100%  1024.41kB  4.82%  runtime.malg
         0     0%   100%   512.56kB  2.41%  runtime.mcall
         0     0%   100%  1537.69kB  7.24%  runtime.mstart
         0     0%   100%  1537.69kB  7.24%  runtime.mstart0
         0     0%   100%  1537.69kB  7.24%  runtime.mstart1
         0     0%   100%  2050.25kB  9.65%  runtime.newm
         0     0%   100%  1024.41kB  4.82%  runtime.newproc.func1
         0     0%   100%  1024.41kB  4.82%  runtime.newproc1
```
从结果中可见 `main.heaps.func1` 占用了 85.53% 的内存，我们接下来直接使用 `list` 命令查看具体是谁占有的：
```bash
(pprof) list main.heaps.func1
Total: 20.75MB
ROUTINE ======================== main.heaps.func1 in /Users/jayson/workspace/go/pprof/main.go
   17.75MB    17.75MB (flat, cum) 85.53% of Total
         .          .     36:		for i := 0; i < 100; i++ {
         .          .     37:			wg.Add(1)
         .          .     38:			go func() {
         .          .     39:				defer wg.Done()
         .          .     40:
   17.75MB    17.75MB     41:				bs := make([]byte, 1024*rand.Int63n(1024))
         .          .     42:				time.Sleep(time.Millisecond * time.Duration(rand.Int63n(1000)))
         .          .     43:				_, _ = fmt.Fprintf(io.Discard, "malloc %d bytes\n", len(bs))
         .          .     44:			}()
         .          .     45:		}
         .          .     46:
```
现在就可以非常简单的看到是 `bs` 这个变量持有了 `17.75MB` 的内存，结合业务逻辑就可以综合分析这是否正常，是否需要进行优化了。

#### 协程栈分析
如果怀疑程序中可能存在协程泄漏，那我们可以通过以下命令检查协程的状态是否正常：
```bash
$ go tool pprof http://localhost:8088/debug/pprof/goroutine
Fetching profile over HTTP from http://localhost:8088/debug/pprof/goroutine
Type: goroutine
Time: Feb 11, 2023 at 3:22pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```
与堆内存分析一样，我们直接通过 `top` 命令查看当前协程主要阻塞在哪些调用上：
```bash
(pprof) top
Showing nodes accounting for 181, 99.45% of 182 total
Showing top 10 nodes out of 35
      flat  flat%   sum%        cum   cum%
       180 98.90% 98.90%        180 98.90%  runtime.gopark
         1  0.55% 99.45%          1  0.55%  runtime.goroutineProfileWithLabels
         0     0% 99.45%          1  0.55%  internal/poll.(*FD).Accept
         0     0% 99.45%          1  0.55%  internal/poll.(*pollDesc).wait
         0     0% 99.45%          1  0.55%  internal/poll.(*pollDesc).waitRead (inline)
         0     0% 99.45%          1  0.55%  internal/poll.runtime_pollWait
         0     0% 99.45%          1  0.55%  main.costly
         0     0% 99.45%          1  0.55%  main.goroutines
         0     0% 99.45%         80 43.96%  main.goroutines.func1
         0     0% 99.45%          1  0.55%  main.heaps
(pprof) top -cum
Showing nodes accounting for 180, 98.90% of 182 total
Showing top 10 nodes out of 35
      flat  flat%   sum%        cum   cum%
       180 98.90% 98.90%        180 98.90%  runtime.gopark
         0     0% 98.90%        177 97.25%  time.Sleep
         0     0% 98.90%         96 52.75%  main.heaps.func1
         0     0% 98.90%         80 43.96%  main.goroutines.func1
         0     0% 98.90%          2  1.10%  runtime.goparkunlock (inline)
         0     0% 98.90%          2  1.10%  runtime.semacquire1
         0     0% 98.90%          2  1.10%  sync.(*WaitGroup).Wait
         0     0% 98.90%          2  1.10%  sync.runtime_Semacquire
         0     0% 98.90%          1  0.55%  internal/poll.(*FD).Accept
         0     0% 98.90%          1  0.55%  internal/poll.(*pollDesc).wait
```
可以看到基本所有的协程都阻塞在 `runtime.gopark` 函数上，而且是从 `time.Sleep` 上进入的阻塞状态，我们可以通过 `tree` 命令来证明这一点：
```bash
(pprof) tree
Showing nodes accounting for 181, 99.45% of 182 total
----------------------------------------------------------+-------------
      flat  flat%   sum%        cum   cum%   calls calls% + context
----------------------------------------------------------+-------------
                                               177 98.33% |   time.Sleep
                                                 2  1.11% |   runtime.goparkunlock
                                                 1  0.56% |   runtime.netpollblock
       180 98.90% 98.90%        180 98.90%                | runtime.gopark
----------------------------------------------------------+-------------
                                                 1   100% |   runtime/pprof.runtime_goroutineProfileWithLabels
         1  0.55% 99.45%          1  0.55%                | runtime.goroutineProfileWithLabels
----------------------------------------------------------+-------------
... 省略后续内容 ...
```

#### CPU 占用分析
如果我们需要分析当前是否有性能问题，或者检查某段代码是否可以进行改进优化，我们则可以使用以下命令对程序的 CPU 使用量进行分析：
```go
$ go tool pprof "http://localhost:8088/debug/pprof/profile?seconds=30"
Fetching profile over HTTP from http://localhost:8088/debug/pprof/profile?seconds=30
Type: cpu
Time: Feb 11, 2023 at 3:36pm (CST)
Duration: 30.18s, Total samples = 8.06s (26.70%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```
需要注意的是，因为需要获取 CPU 的占用率所以需要在对一段时间进行采样，这是通过在参数中指定 `seconds` 参数实现的。如之前的分析一样，我们直接通过 `top` 命令查看具体占用分析：
```bash
(pprof) top
Showing nodes accounting for 7.90s, 98.01% of 8.06s total
Dropped 37 nodes (cum <= 0.04s)
Showing top 10 nodes out of 57
      flat  flat%   sum%        cum   cum%
     6.44s 79.90% 79.90%      6.70s 83.13%  main.costly
     0.35s  4.34% 84.24%      0.48s  5.96%  runtime.kevent
     0.22s  2.73% 86.97%      0.22s  2.73%  runtime.asyncPreempt
     0.20s  2.48% 89.45%      0.21s  2.61%  runtime.madvise
     0.18s  2.23% 91.69%      0.18s  2.23%  runtime.libcCall
     0.15s  1.86% 93.55%      0.19s  2.36%  runtime.pthread_cond_wait
     0.12s  1.49% 95.04%      0.12s  1.49%  runtime.usleep
     0.11s  1.36% 96.40%      0.11s  1.36%  runtime.memclrNoHeapPointers
     0.07s  0.87% 97.27%      0.07s  0.87%  runtime.pthread_kill
     0.06s  0.74% 98.01%      0.06s  0.74%  runtime.pthread_cond_signal
```
从打印结果可见，79.90% 的 CPU 时间都消耗在 `main.costly` 方法上。我们可以通过 `list` 命令来查看具体的代码行，根据业务确定是否有优化空间：
```bash
(pprof) list main.costly
Total: 8.06s
ROUTINE ======================== main.costly in /Users/jayson/workspace/go/pprof/main.go
     6.44s      6.70s (flat, cum) 83.13% of Total
         .          .     48:	}
         .          .     49:}
         .          .     50:
         .          .     51:func costly() {
         .          .     52:	for {
     6.44s      6.66s     53:		for i := 0; i < 1_000_000_007; i++ {
         .          .     54:		}
         .       40ms     55:		time.Sleep(time.Second)
         .          .     56:	}
         .          .     57:}
         .          .     58:
         .          .     59:func main() {
         .          .     60:	go heaps()
```

#### 程序运行时事件分析
运行时的事件分析与之前的指标分析有所不同，trace 可以提供指定时间内程序发生的事件的统计信息。而之前的单独项目的性能分析是站在局部视角上，trace 则是站在全局视角对程序进行分析。<br />我们通过如下命令来下载一段时间内的事件统计信息，并通过 `go tool trace` 打开浏览器查看指标数据：
```shell
$ curl -o trace.out "http://localhost:8088/debug/pprof/trace?seconds=30"
$ go tool trace trace.out
```
点击 `View trace` 之后就可以看到如下界面：
![trace](assets/trace.png)


### 通过图形化的方式对性能指标进行分析
除了以上命令行形式之外，我们还可以通过在 `pprof` 命令中增加参数 `-http ip:port` 来以图形化界面的方式对各项指标进行分析。
```bash
$ go tool pprof -http :8081 http://localhost:8088/debug/pprof/heap
$ go tool pprof -http :8081 http://localhost:8088/debug/pprof/goroutine
$ go tool pprof -http :8081 "http://localhost:8088/debug/pprof/profile?seconds=30"
```

#### 火焰图
火焰图可以快速准确地识别出最频繁使用的代码路径，从而得知程序的瓶颈所在。在图形化界面上，我们可以通过点击 `VIEW -> Flame Graph` 来查看：
![Flame Graph](assets/flame-graph.png)


### 参考文献

- [Profiling Go Programs](https://blog.golang.org/profiling-go-programs)

