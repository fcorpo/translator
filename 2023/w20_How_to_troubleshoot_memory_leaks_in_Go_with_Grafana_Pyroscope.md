
# 如何用 Grafana Pyroscope 解决 Go 中的内存泄漏问题  

- 原文地址：https://grafana.com/blog/2023/04/19/how-to-troubleshoot-memory-leaks-in-go-with-grafana-pyroscope/
- 原文作者：[Cyril Tovena](https://grafana.com/author/cyril/)
- 本文永久链接：https://github.com/gocn/translator/blob/master/2023/w20_How_to_troubleshoot_memory_leaks_in_Go_with_Grafana_Pyroscope
- 译者：[sijing233](https://github.com/sijing233)
- 校对：[lsj1342](https://github.com/lsj1342)

------

在任何编程语言中，内存泄露都是一个严重的问题，Go 也不例外。虽然 Go 有内存回收机制，但是也容易受到内存泄露的影响，从而导致性能下降、操作系统内存耗尽。

为了防止这样的情况发生，Linux 操作系统实现了一种保护机制 OOM Killer (Out-of-Memory)，这种机制会识别出并终止那些占用内存过多和导致操作系统无法响应的进程。

在这篇博文中，我们将探讨 Go 语言中，最常见的内存泄露原因，并演示如何使用 [Grafana Pyroscope](https://grafana.com/blog/2023/03/15/pyroscope-grafana-phlare-join-for-oss-continuous-profiling/?pg=blog&plcmt=body-txt)，一个开源的持续分析解决方案，来发现和修复这些漏洞。

## 传统的可观测信号

一般来说，通过长期监视程序或系统的内存使用情况，可以检测到内存泄露。

![A Grafana dashboard displays memory leak.](https://github.com/gocn/translator/raw/master/static/images/2023/w20/go-memory-leak-pyroscope-1.png)

*通过在 Grafana Cloud 中使用 Kubernets 监控可以看到内存泄露.*

如今，由于系统的复杂性，很难确定代码中发生内存泄漏的位置。 然而，这些泄漏可能会导致严重的问题:  

1. **会降低性能。** 随着内存泄漏的发生，系统可用内存越来越少，这可能导致程序变慢或崩溃，从而导致性能下降。  
2. **系统不稳定。** 如果内存泄漏足够严重，可能会导致整个系统变得不稳定，导致崩溃或其他系统故障。  
3. **增加成本和资源使用。** 当内存泄漏发生时，系统可能会使用更多的资源来管理内存，这会降低系统资源对其他程序的总体可用性。  

 

由于这些原因，尽快发现并修复它们是很重要的。 

## Go 中内存泄漏的常见原因

开发人员通常会因未能正确关闭资源或未能避免无限创建资源而造成内存泄漏，这也适用于 goroutines 协程。 Goroutines 协程可以被认为是一种资源，因为它们会消耗系统资源（例如内存和 CPU 时间），如果管理不当，可能就会导致内存泄漏。

Go 协程是由 Go 运行时管理的轻量级执行线程，它们可以在 Go 程序执行期间动态创建和销毁。  

理论上来说，在 Go 中我们可以创建无数个协程，GO 运行时可以创建和管理数百万个协程，而不会产生显著的性能开销。但实际上，系统资源是固定的（如内存、CPU 和 I/O 资源），协程的限制由可用的系统资源决定。

开发人员在使用协程时，经常创建了多个协程而没有正确的管理它们的生命周期。对于未使用的协程，即使不需要它们，也会继续消耗系统资源，这可能会导致内存泄露。

如果一个协程被创建而没有关闭，它可以无限期地继续执行，并在内存中保存对对象的引用，防止它们被[垃圾收集](https://tip.golang.org/doc/gc-guide)。 这可能导致程序的内存使用量随着时间的推移而增长，从而不能被垃圾回收。  

在希望并行处理单个 HTTP 请求所完成的工作的情况下，创建多个协程并将工作分派给它们是合理的。 

```go
package main

import (
	"log"
	"net/http"
	"time"

	_ "net/http/pprof"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		responses := make(chan []byte)
		go longRunningTask(responses)
           // do some other tasks in parallel
	})
	log.Fatal(http.ListenAndServe(":8081", nil))
}

func longRunningTask(responses chan []byte) {
	// fetch data from database
	res := make([]byte, 100000)
	time.Sleep(500 * time.Millisecond)
	responses <- res
}
```

上面的这段简单的代码，在连接到并行数据库的 HTT 服务器过程中，造成了内存泄露。因为 HTTP 请求不等待响应 channel，所以 longRunningTask 永远处于阻塞状态，从而泄露了用它创建的协程和资源。  

为了防止这种情况的发生，我们需要正确终止协程。我们可以使用各种技术来完成，比如使用 channel 来通知什么时候应该退出协程，或是使用 `context.Cancel` 将取消信号传播到协程，或是使用 `sync.WaitGroup` 确保所有协程在退出程序之前都已完成。

为了避免无限地创建协程，我还建议使用[工作池](https://gobyexample.com/worker-pools)。 当应用程序处于压力之下时，创建太多的协程可能会导致性能下降，因为 Go 运行时必须管理它们的生命周期。  

 

Go 中另一个常见的协程和资源泄漏的情况是没有正确释放 Timer 或 Ticker。 Go 文档 [regarding the time.After function](https://pkg.go.dev/time#After) 表明了这种可能性:  

> " After 等待持续时间结束，然后在返回的通道上发送当前时间。 它相当于 NewTimer(d).C。 在计时器触发之前，垃圾收集器不会恢复底层计时器。 如果关注效率，则使用 NewTimer 并调用 Timer。 如果不再需要计时器，则停止 ”  

 

如下所示，我建议您在坚持在协程中使用 [timer.NewTimer](https://pkg.go.dev/time#NewTimer) 和 [timer.NewTicker](https://pkg.go.dev/time#NewTicker) 时始终坚持使用它们，以便在请求结束时正确释放资源。 

## 如何用 Pyroscope 发现和修复内存泄露

持续分析可能是查找内存泄漏的一种有用方法，特别是在内存泄漏发生很长一段时间或发生得太快而无法手动观察的情况下。 持续分析包括定期采样程序的内存和协程的使用情况，以识别可能指示内存泄漏的模式和异常。  

通过分析内存和协程使用情况，您可以识别 Go 应用程序中的内存泄漏。 以下是使用 Pyroscope 的步骤。  

(注:虽然这篇博文主要关注 Go 语言，但 Pyroscope 也支持其他语言的内存分析。) 

### 步骤 1:确定内存泄漏的来源  

假设您已经进行了监控，第一步是使用日志、度量或跟踪找出系统哪个部分有问题

这可以通过多种方式表现出来:  

- 应用程序或 Kubernetes 重新启动日志
- 应用程序或主机内存使用情况告警
- 典型的 SLO 缺口

一旦您确定了系统的部分和时间，您就可以使用持续分析来确定有问题的功能

### 步骤 2:将 Pyroscope 与您的应用程序集成

要开始分析 Go 应用程序，你需要在应用程序中包含我们的 Go 模块: 

```go
go get github.com/pyroscope-io/client/pyroscope
```

然后将以下代码添加到应用程序中: 

```go
package main

import "github.com/pyroscope-io/client/pyroscope"

func main() {
    // These 2 lines are only required if you're using mutex or block profiling
    // Read the explanation below for how to set these rates:
    runtime.SetMutexProfileFraction(5)
    runtime.SetBlockProfileRate(5)

    pyroscope.Start(pyroscope.Config {
        ApplicationName: "simple.golang.app",

        // replace this with the address of pyroscope server
        ServerAddress: "http://pyroscope-server:4040",

        // you can disable logging by setting this to nil
        Logger: pyroscope.StandardLogger,

        // optionally, if authentication is enabled, specify the API key:
        // AuthToken:    os.Getenv("PYROSCOPE_AUTH_TOKEN"),

        // you can provide static tags via a map:
        Tags: map[string] string {
            "hostname": os.Getenv("HOSTNAME")
        },

        ProfileTypes: [] pyroscope.ProfileType {
            // these profile types are enabled by default:
            pyroscope.ProfileCPU,
                pyroscope.ProfileAllocObjects,
                pyroscope.ProfileAllocSpace,
                pyroscope.ProfileInuseObjects,
                pyroscope.ProfileInuseSpace,

                // these profile types are optional:
                pyroscope.ProfileGoroutines,
                pyroscope.ProfileMutexCount,
                pyroscope.ProfileMutexDuration,
                pyroscope.ProfileBlockCount,
                pyroscope.ProfileBlockDuration,
        },
    })

    // your code goes here
}
```

要了解更多信息，请查看 [Pyroscope 文档](https://pyroscope.io/docs/golang/)。 

### 步骤 3:下钻关联指标

我建议你先看看 goroutines 随时间推移，是否有任何值得关注的问题，然后切换到内存调查。

![](https://github.com/gocn/translator/raw/master/static/images/2023/w20/huoyantu.png)
```html
<iframe frameborder="0" width="100%" height="400" src="https://flamegraph.com/share/dee1210a-ddff-11ed-9b0d-d641223b6af4/iframe?colorMode=light&amp;onlyDisplay=flamegraph&amp;showToolbar=true" style="box-sizing: border-box;"></iframe>
```

在这种情况下，很明显我们的 `longRunningTask` 是问题所在，我们应该看看这个。在实际使用中，您必须探索并将您在火焰图上看到的内容与您对应用程序的预期联系起来。

有趣的是，火焰图中的 goroutine 堆栈跟踪，实际上显示了函数的当前状态——在我们的示例中，它被阻止发送到 channel。这是一条可以帮助您理解代码正在做什么的信息。

> 对于内存，已分配内存的函数以及分配了多少内存，但不会显示谁在保留它。这依靠您来找出代码中错误地保留了内存的位置。

要了解更多关于 Go 的性能分析，我建议您阅读 [Julia Evans 的这篇很棒的博客文章](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)。

### 第四步:通过测试确认和预防

假设您现在已经确定了问题所在，您可能想要立即修复它—但是我建议您首先编写一个测试来展示问题。  

这样就可以避免其他工程师再次犯同样的错误。 既然你确信你确实找到了问题，你就会有一个可靠的反馈循环来证明它实际上已经解决了。  

Go 有一个强大的测试框架，您可以使用它来编写基准测试或测试来重现您的场景。  

在[基准测试](https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go)期间，您甚至可以使用 `-benchmem`.

```go
go test -bench=. -benchmem
```

如果需要，您还可以使用 [runtime.ReadMemStats](https://pkg.go.dev/runtime#ReadMemStats) 编写一些自定义逻辑来输出内存分配。  

您还可以使用 [goleak](https://github.com/uber-go/goleak) 包来验证在程序执行后没有协程泄漏。

```go
func TestA(t *testing.T) {
	defer goleak.VerifyNone(t)

	// test logic here.
}
```



### 步骤 5:修复内存泄漏

既然您可以重现并理解您的问题，那么是时候迭代修复并部署它进行测试了。您可以再次利用持续分析来监控您的变更并确认符合您的预期。

## 了解 Pyroscopet

最后，在所有这些调查和工作之后，您可以通过分享 [Grafana] 内存使用下降的截图 (https://grafana.com/grafana/?pg=blog&plcmt=body-txt) 与您的同事分享您的胜利。  

如果你以前没有尝试过 [Pyroscope](http://pyroscope.io/)，这是一个开始的好时机。 现在，Pyroscope 团队是 Grafana 实验室的一部分，我们正在将这个项目与 Grafana Phlare 合并。 查看我们发布消息的[博客文章](https://grafana.com/blog/2023/03/15/pyroscope-grafana-phlare-join-for-oss-continuous-profiling/?pg=blog&plcmt=body-txt)，以及[在 GitHub 上关注进展](https://github.com/grafana/pyroscope)。  

想了解更多? [现在注册](https://grafana.com/about/events/grafanacon/2023/?pg=blog&plcmt=button)免费参加 GrafanaCon 2023，并直接听取 Grafana Pyroscope 项目负责人的会议，[持续分析与 Grafana Pyroscope: 开发人员经验，火焰图，以及更多](https://grafana.com/about/events/grafanacon/2023/session/continuous-profiling-with-grafana-pyroscope/?pg=blog&plcmt=body-txt).) 
