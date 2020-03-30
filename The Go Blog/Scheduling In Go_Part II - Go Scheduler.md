原文：https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html

#### 1、介绍

在本系列的[第一部分](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)，我解释重要的操作系统调度的知识，它对于理解go的调度很重要。
这一节中，我会在semantic级别解释GO的调度，并且会关注其high-level的行为。
Go的调度是一个复杂的系统，它的一些小的细节并不重要。
最重要的是，我们需要对它的运行机制和行为有很好的建模。这有助于你更好地做出工程决定。

#### 2、你的程序如何开始 

当你的go程序开始时，它会被分配一个逻辑的处理器（P）给每一个虚拟的核，它对于host machine是唯一的。
如果你的处理器的每个物理核有多个硬件线程([Hyper-Threading](https://en.wikipedia.org/wiki/Hyper-threading))，则每个硬件线程对你的go程序来说就相当于一个虚拟核。
为了更好地理解，我们来看下我的MacBook Pro的配置。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure1.png)

你可以看到，我的处理器有4个物理核。这个并不是每个物理核的硬件线程数。
对于Intel Core i7处理器来说，它有Hyper-Threading，这意味着每个物理核有2个硬件线程。
它会告诉go程序，有8个虚拟核可以并行地执行OS线程。

为了验证这个，我们可以使用下列的程序：

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {

    // NumCPU returns the number of logical
    // CPUs usable by the current process.
    fmt.Println(runtime.NumCPU())
}
```

当我在本地运行这个程序时，NumCPU()返回的结果是8。
因此，我在本地上运行的任何go程序都会分配8个P。

每一个P都会分配一个OS线程（M）。M表示机器（machine）。
这个线程由OS管理，OS会提供这个线程在核上的执行。
这意味着，当我在本地运行一个go程序，我有8个线程执行我的作业，每一个线程都跟一个P挂钩。

每一个Go程序都会有一个初始的goroutine（G），它是go程序执行的路径。
一个goroutine实际上是[协程](https://en.wikipedia.org/wiki/Coroutine)，不过在go里，我们管它叫goroutine。
你可以认为goroutine是一个应用级别的线程，它跟OS线程很多方面都相似。
正如OS线程是在核上进行上下文的切换，goroutine则是在M上进行上下文切换。

go调度的最后一块拼图是运行队列（run queue）。
在go调度中，有两个不同的运行队列：全局队列（GRQ）和本地队列（LRQ）。
每一个P都会分配一个LRQ，它管理分配给P的goroutine的执行。
这些goroutine都在P分配的M上进行上下文的切换。
GRQ并没有分配到特定的P。有一个处理过程来把goroutine从GRQ移动到LRQ。我们后面会进行讨论。

下图展示了所有部件之间的关系。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure2.png)

#### 3、协作调度 

正如我在前一篇文章讨论的，OS调度是抢占式调度。本质上这意味着你不能预测调度器在任何时间会做什么。
这由内核来做决定，而这是不确定的（non-deterministic）。
在OS上运行的应用管不到内核的调度，除非它们用同步的原语，例如[atomic指令](https://en.wikipedia.org/wiki/Linearizability)和[mutex调用](https://en.wikipedia.org/wiki/Lock_(computer_science)。

go调度是go runtime的一部分，go runtime则内置在你的应用中。
这意味这go调度运行在[用户空间](https://en.wikipedia.org/wiki/User_space)，也就是内核之上。
当前go调度并不是抢占式调度，而是协助式调度([cooperation scheduler](https://en.wikipedia.org/wiki/Cooperative_multitasking))。
协助式调度意味着调度器需要定义用户空间的事件，并在代码的安全节点做出调度决定。

go的协助调度最聪明的一点是，它看起来像是抢占式调度。
你不能预测go scheduler会做什么。因为决定并没有交到开发者手上，而是由go runtime来完成。
因此，认为go调度是抢占式调度很重要，因为调度器是non-deterministic，而且没有太多扩展。

#### 4、goroutine的状态 

像线程一样，goroutine同样有3个high-level的状态。
这是go调度给任何goroutine分配的角色。一个goroutine只能是三个状态之一：等待，可运行和执行中。

等待(waiting)：这意味着goroutine停止，并等待能够让它继续的东西。这可能是等待操作系统调用，或者同步调用等。
(这些类型的延迟)[https://en.wikipedia.org/wiki/Latency_(engineering)]是影响系统性能的根源。

可运行(runnable)：这意味goroutine需要M的时间，这样它可以执行分配给它的指令。
如果你有很多gorouine在等待时间，则这个goroutine就需要等很久。
同样的，一个goroutine如果有大量的gorouine竞争时间，它获得的时间就越少。
这一类的调度延迟也会影响性能。

执行中(executing)：这意味着goroutine要安置在一个M上，并执行它的指令。

#### 4、上下文切换

go调度需要很好地定义用户空间的事件，这样可以在代码的安全点去进行上下文切换。
这些事件和安全点是通过函数调用来表示的。
函数调用对go调度的良好运行至关重要。如今(GO 1.11)，如果你运行tight loop而不进行函数调用，你可能会因为调度和垃圾回收导致延迟。
让函数调用发生在合理的时间，这很重要。

注意：这个1.12的[提议](https://github.com/golang/go/issues/24543)，应用在go调度里引用非协助式的技术（non-cooperative preemption techniques）来允许在tight loop中抢占。

你的go程序中，有4中事件会导致调度器做出决定。
这并不意味着它总会发生，只是说调度器获得了机会：

* 使用了关键字go 

* 垃圾回收

* 系统调用

* 同步和编排治理（Synchronization and Orchestration）

##### 使用关键字go 

使用关键字go是创建goroutine。一旦新的goroutine创建，调度器会获得机会做出调度决定。

##### 垃圾回收

由于GC运行它们自己的goroutine集，这些goroutine需要M的时间来运行。
这会使得GC产生很多调度的混乱。不过，调度器很聪明，它知道goroutine在做什么，因而会权衡做出决定。
一个聪明的决定是，在GC阶段，一个goroutine要接触其他goroutine没有接触的堆时，需要上下文切换。
在GC运行时，它会做很多调度的决定。

##### 系统调用

一个goroutine做系统调用时，它会导致goroutine阻塞M，有时调度器会通过上下文切换把这个goroutine换出M，并换一个新的goroutine到同一个M上。
然而，有时需要一个新的M来继续执行P中排队的goroutine。
在下一节中会解释具体细节。

##### 同步和编排治理

如果一个atomic、mutex或者channel导致goroutine阻塞，则调度器会通过上下文切换来执行一个新的goroutine。
直到这个goroutine能重新运行，它会重新排队，并最终通过上下文切换回到M。

#### 5、异步系统调用

当你的OS能使用异步系统调用时，有时可以使用[network poller](https://golang.org/src/runtime/netpoll.go)来使得其更加高效。
在不同系统，它们实现技术不一样：kqueue(MacOs)、epoll(Linux)和iocp(Windows)。

我们现在使用的很多操作系桶都能处理基于网络的系统调用。
这也是network poller名字的由来，它主要的用途是处理网络操作。
通过network poller处理网络系统调用，调度器能够避免goroutine在运行系统调用时阻塞M。
这可以让M执行P的LRQ队列里其他的goroutine，而不需要创建新的M。
这减轻了OS的调度的压力。

了解其工作方式的最好方法就是通过例子：

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure3.png)

上图是基本的调度图。goroutine-1在M上执行，还有3个goroutine在LRQ上等待。
network poller在等待，没有任何事情做。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure4.png)

在上图中，goroutine-1需要进行网络系统调用，因而它移动到network poller，同时异步网络调用开始处理。
当goroutine-1系统到network poller，M则能执行LRQ中的goroutine。
在这个case中，goroutine-2上下文切换到M。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure5.png)

上图中，异步系统调用通过network poller完成，而goroutine-1则重新回到LRQ。
当goroutine-1上下文切换回M，跟它相关的go代码则可以重新执行。
这里最大的优势是，执行网络系统调用，不需要额外的M。
network poller拥有OS线程，它一直处在有效的事件循环（event loop）中。

#### 6、同步系统调用

如果goroutine想用进行系统调用，而不能异步完成，会怎么样？
在这个case中，不能使用network poller，而goroutine进行系统调用会阻塞M。
这在现实中总是会发生。一个不能使用异步方式的系统调用是基于文件的系统调用（file-based system call）.
如果你使用CGO，那么在调用C函数时也会阻塞M。

注意：Windows OS能够异步进行基于文件的系统调用。技术上，当运行在Windows系统时，可以使用network poller。

我们来看下阻塞M的同步系统调用会发生什么。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure6.png)

上图还是一个基本的调度图，但这次goroutine-1将要进行同步系统调用，并阻塞M1。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure7.png)

在上图中，调度器能够识别出goroutine-1导致M阻塞。
这时，会把M1和P分离，而阻塞的goroutine-1仍然会在M1上。
这时，调度器会引入新的M2来服务P。接着，goroutine-2会从LRQ中取出，并在M2上做上下文切换。
如果M已经存在，则这个切换会比创建新的M更快。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure8.png)

在上图中，由goroutine-1产生的系统调用完成。
这时，goroutine-1会回到LRQ，由P来服务。
M1则会放到一边，以便后续使用。

#### 7、工作窃取（work stealing）

调度器的另一个方面是，它是一个work-stealing的调度。它在一些领域能够保证调度效率。
你最不希望的就是M进入等待状态，因为OS会把M切换出内核。
这意味着，P不能做任何工作，即便是有goroutine在可运行的状态，它也要等到M切换回内核。
work stealing会帮助平衡所有P之间的goroutine，使得其在分配和执行上更加高效。

我们通过例子来分析。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure9.png)

在上图中，我们有一个多线程的go程序，有2个P，每个P都有4个goroutine，此外还有一个goroutine在GRQ。
如果1个P能很快完成它的goroutine工作，会发生什么？

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure10.png)

在上图中，P1没有更多的goroutine执行。
但还有一些等待执行的goroutine在P2的LRQ和GRQ。
这时，P1就需要[stealing work](https://golang.org/src/runtime/proc.go)，其具体规则如下：

```go
runtime.schedule() {
    // only 1/61 of the time, check the global runnable queue for a G.
    // if not found, check the local queue.
    // if not found,
    //     try to steal from other Ps.
    //     if not, check the global runnable queue.
    //     if not found, poll network.
}
```

基于上面列举的规则，P1需要检验P2的goroutine，并拿走一半。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure11.png)

上图中，P2一半的goroutine被拿到了P1，这时P1可以执行这些goroutine了。

如果P2执行完了它所有的P1，而P1的LRQ队列里没有多余的goroutine了呢？

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure12.png)

上图中，P2完成了所有的工作，它需要去"窃取"一些。
首先它会查看P1的LRQ，但没有发现任何任务。
接下来它会查看GRQ。它会找到goroutine-9。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure13.png)

上图中，P2会从GRQ窃取goroutine-9，并开始执行。
对于任务窃取来说，它的一个好处就是保持M一直运行，而不会进入idle状态。
这个work stealing可以在内部可以认为是M的自旋。
这样的自旋还有其他好处，而JBD在她的(博客)[https://rakyll.org/scheduler/]里有解释。

#### 8、实践案例 

有了相应的机制和语义，接来下会展示如何把他们结合在一起，让go调度能处理更多的任务。
假设一个多线程的应用，它是由C写的，它管理两个OS线程，并在彼此之间传递消息。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure14.png)

上图中，有两个线程在传递消息。线程1在核1上进行上下文切换，并允许线程1发送消息给线程2。

注意：消息怎么发送的不重要。重要的是，线程的状态。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure15.png)

上图中，当线程1完成发送消息后，它需要等待应答。
这会导致线程1切换出核1，并转移到等待状态。
而线程2会得到消息的通知，它会进入可执行的状态。
现在，OS可以进行上下文切换，并把线程2放到核2上执行。
接下来，线程2处理消息，并发送新的消息给线程1。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure16.png)

上图中，在线程2上会再次切换线程上下文。现在，线程2会从执行状态切换到等待状态，而线程1会从等待状态到可执行状态，并最终进入可运行状态，这会让他进行处理，并回复消息。

这些上下文切换和状态变更需要时间，这会限制我们工作的速度。
由于每个上下文切换会出现大约1000纳秒的延迟，每纳秒会执行12条指令，这样大约有12k的指令在此期间没有执行。
由于这些线程会在不同核之间切换，因而会导致缓存没有命中，进而会有更多延时。

我们用同样的例子，但是从goroutine和go调度的视角来看。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure17.png)

上图有2个goroutine，他们之间相互发送消息。
G1在M1上进行上下文切换，它在核1上运行。
因此，G1可以发送消息给G2。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure18.png)

上图中，G1完成发送消息，它现在要等待回应。
这会使得G1从M1上切换出去，进入等待状态。
而G2获得了消息通知，因而它进入可执行状态。
这时go调度可以进行上下文切换，使得G2在M1上执行，而M1是在核1上运行。
接下来，G2处理了消息，并发送新的消息给G1。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler_figure19.png)

上图中，G1接收到G2的消息后会再进行一次上下文切换。
现在G2从执行状态切换为等待状态，而G1从等待状态进入可执行状态，最终进入执行状态，这样它会进行处理，会发送新的消息。

从表面上看，似乎事情并没有不同。
无论是线程还是goroutine，都会发生上下文切换和状态变更。
然而，使用线程和goroutine有一个最大的不同，而起初看上去并不明显。

在使用goroutine时，整个过程中，都使用相同的OS线程和核心。
这意味着，在OS视角，OS线程并没有转移到等待状态。
这样，我们在线程切换时造成的指令丢失，在goroutine上并不会发生。

本质上，go已经把IO阻塞变成OS级别的CPU-bound。
由于所有的上下文切换都在应用层，我们不会丢失大约12k的指令。
在go中，这些上下文切换大约耗费200纳秒，大约2.4k条指令。
这些调度同样提供cache-line效率。
这就是为什么我们不需要比虚拟内核更多的线程。
在Go里，随着时间推移，其实可以做更多的工作，因为go调度尝试使用更少的线程，而在每个线程上做更多的事情，这样有助于降低OS和硬件的负载。

#### 9、结论

go调度在设计的时候就考虑了OS和硬件的复杂性。
在OS级别，把IO阻塞转化为CPU-bound，这也就是我们能够最大利用CPU的地方。
这也是为什么不需要比虚拟核更多的线程。
你可以认为，每个虚拟核上只要有一个OS线程就能完成所有工作。
这样对于网络app或者其他app可以不对OS线程造成阻塞。

对于开发者，你仍然需要理解你的app在做什么。
不能不限制地创建goroutine，并期望有很好的性能。
Less is always more，但是理解这些go调度语义，你能够更好地做出工程决策。
在下一章中，我会将会讨论以保守方式利用并发以获得更好的性能，当然，也会平衡你需要添加代码的复杂度。







