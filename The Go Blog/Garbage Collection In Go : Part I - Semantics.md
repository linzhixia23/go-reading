原文：https://www.ardanlabs.com/blog/2018/12/garbage-collection-in-go-part1-semantics.html

#### 1、介绍

gc负责跟踪堆的内存分配，释放不需要的内存，保留正在使用的内存。编程语言做出实现gc的决定是很复杂的，它并不要求开发了解这些细节。
而且，编程语言不同的VM或者runtime版本，gc的实现总是在变化和演进。
对于应用开发者来说，重要的是了解这门语言的gc原理，以及如何理解它的行为而不要考虑它的具体实现。

在go的1.12版本中，go编程语言实现了无分代并发三色编辑清除回收器（non-generational concurrent tri-color）。
如果你想知道标记清除回收器如何做的，[这篇文章](https://spin.atomicobject.com/2014/09/03/visualizing-garbage-collection-algorithms/)有一个很好的动画。
go的回收器在每个release版本都会演化。因此任何讲实现细节的文章，在下一个新的版本出来之后，它都变成不够准确。

也就是说，我不会关注实际的实现细节。我会关注你使用时的体验，以及你期望看到的行为。
这篇文章里，我会跟你分享回收器的行为，并解释这些行为，而不管它当前如何实现，以及之后会如何变化。
这会让你成为一个更好的go开发者。

在这里，你可以找到关于[gc的资料](https://github.com/ardanlabs/gotraining/tree/master/reading#garbage-collection)

#### 2、堆不是容器

我从来不会认为堆是一个你可以取和释放东西的容器。
要理解没有线性限制的内存都定义为"堆"。
任何在进程空间中保留的应用使用的内存，都可以通过堆来分配。
至于它是虚拟存储还是物理存储，都跟我们的模型无关。理解这个会帮助你更好的理解gc。

#### 3、回收器的行为

当回收开始时，回收器会进行三个阶段的工作。有两个会导致STW延迟，另一个则会降低应用的吞吐量。
这三个阶段是：

* 标记开始 - STW 

* 标记进行 - 并发

* 标记结束 - STW 

下面仔细看每一个阶段：

##### 标记开始 - STW 

当一个回收器开始时，第一个要做的事情是，打开写屏障。
写屏障的作用是允许回收器在回收阶段保持堆上数据的完整性，因为回收器和应用的goroutine都在并发地执行。

为了打开写屏障，每一个应用的goroutine必须停止。
这个动作必须非常快，平均大概是10-30微秒。这是在应用程序的goroutine正常的情况下。

注意：为了更好地理解调度器，最好保证读过[go scheduler系列](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)的文章。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure01.png)

上图展示了4个goroutine在回收开始前的运行。
这4个goroutine必须停止。唯一能做到的方式是，回收器观察和等待每个goroutine进行函数调用。
函数调用能保证goroutine达到一个安全的停止的点。
如果这些goroutine，一些进行函数调用，而另一个没有，会怎么样呢？ 

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure02.png)

上图展示了一个真实的问题。回收器不能开始，除非运行在P4上的goroutine停止，然而这不会发生，因为它运行一个[tight loop](https://github.com/golang/go/issues/10958)的数学运算。

```go
func add(numbers []int) int {
	var v int
	for _, n := range numbers {
		v += n
	}
	return v
}
```

上面是P4运行的代码。
根据slice的size长度，这个goroutine可能会运行不合理的时间，并没有机会停止。
这样的代码会延迟回收器的开始。
更糟糕的是，在回收器等待时，其他的P不能服务其他任何goroutine。
因此，goroutine在一个合理的时间内进行函数调用很重要。

这也是go团队想加入[preemptive技术](https://github.com/golang/go/issues/24543)的原因。

##### 标记进行 - 并发

当写屏障打开时，回收器开始标记阶段。
回收器第一件做的是占用25%的CPU处理能力。回收器使用goroutine去做回收工作，因而也会像应用程序的goroutine一样使用P和M。
这意味着，我们4个线程的go程序，有一个会完全用于回收工作。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure03.png)

上图显示了回收器使用P1进行它的工作。
现在回收器开始了标记阶段。标记阶段包括在堆内存中标记在使用的值。
首先，它会检测栈里所有存活的goroutine，找到堆内存的root指针。
然后，回收器必须从这些root指针开始遍历堆内存图（heap memory graph）。
当标记工作在P1进行时，应用程序可以继续在P2、P3和P4上工作。
这意味着，回收器的影响被最小化到当前CPU能力的25%。

我希望这就结束了，然而不是。如果在收集过程中，如果使用的堆内存达到上限，而P1上的goroutine还没有完成标记，该怎么办？
如果3个应用中有一个的工作导致回收器不能及时完成，该怎么办？
这样的情况下，这种情况下，新的分配必须放慢速度，尤其是那个标记导致无法完成的goroutine。

如果回收器发现它需要减慢分配，它会使用其他的goroutine来协助标记工作。
这叫做标记协助。goroutine花在标记辅助的时间跟它要添加的堆内存成正比。
标记辅助的一个好处是，它有助于快速完成回收。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure04.png)

图4显示了，运行在P3上的goroutine开始标记辅助，帮助回收工作。
希望其他的goroutine不要再加进来了。分配较快的应用可以看到，大多数运行的goroutine在回收过程中，都做了少量的标记辅助。

回收器的一个目标是，消除对标记辅助的需求。如果每次回收需要大量的标记辅助，回收器可能很快就要开始下一次gc了。

##### 标记结束 - STW 

当标记工作完成后，下一个阶段就是标记结束。
在这个阶段，写屏障会被关闭，大量的清理工作会进行，然后计算好下一次的回收目标。
在标记阶段，处于tight loop的goroutine，会导致标记结束-STW延时延长。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure05.png)

上图展示了在标记结束阶段，所有的goroutine都停止了。
这个过程通常是60-90微秒。这个阶段可以不需要STW，但是使用STW，代码会更简洁，而且没必要增加较大的复杂度来获得这么一点收益。

当回收器结束时，每一个P就能够被应用程序的goroutine使用。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure06.png)

上图显示了当回收过程结束后，所有的P都开始处理应用程序了。

##### 清理- 并发

回收完成之后，会有一个清除的动作。
清除是指，回收堆内存中没有被关联为已使用的内存。它会在应用程序的goroutine尝试在堆内存中分配时发生。
清除的延迟被计入堆的内存分配中，与gc的任何延迟都无关。

以下是我的机器的追踪样本，有12个线程可用于执行goroutine。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure07.png)

上图是trace的一个快照。你可以看到回收过程中，12个P中有3个专门用于GC。
你可以看到协程2450、1979和2696用于标记协助，而不执行应用程序的工作。
在回收之后，只有1个P用于GC，并最终执行STW（标记结束）的工作。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure08.png)

上图的玫瑰色线条显示了goroutine执行清理工作，而非应用程序工作的时刻。
这就是goroutine尝试在堆内存分配新值的时刻。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure09.png)

上图显示了一个goroutine进行清理工作时，栈最后的状态。
它调用runtime.mallocgc来在堆内存分配新的值。
runtime.(*mcache).nextFree调用会导致清理工作。一旦堆上没有更多可分配的内存回收，就不会看到nextFree。

刚才描述的行为只会在回收开始和进行时发生。
配置项GC百分比在决定什么时候开始回收上扮演着关键的角色。

#### 4、GC百分比

运行时中有一个配置项叫GC百分比，它通常默认为100。
这个值表示下次回收开始前，能分配多少内存的比例。
GC百分比设置为100，意味着根据gc完成后，标记为存活堆内存的数量，下一次回收必须在堆上添加100%以上的新内存分配，才会启动。

举个例子，假定一次收集之后，有2MB的堆内存在使用。

注意：本文的堆内存使用跟实际中使用的并不一样。
真实场景里，go的堆内存通常是碎片化的，并不像图中这样齐整。图中提供了一种可视化的方法，以帮助你更好地理解堆内存。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure10.png)

上图展示了当回收完成之后的堆内存情况。
由于GC百分比设置为100%，下次回收会在添加>=2MB内存时发生。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure11.png)

上图展示了在使用的堆内存超过2MB。这会触发一次回收。
一个看这个操作的方法是，在每次回收发生时产生GC trace。

#### 5、GC trace 

gc trace可以在运行任何go应用时，通过设置环境变量GODEBUG的gctrace=1来生成。
每次出现收集时，runtime会把GC trace的信息写道stderr。

```
GODEBUG=gctrace=1 ./app

gc 1405 @6.068s 11%: 0.058+1.2+0.083 ms clock, 0.70+2.5/1.5/0+0.99 ms cpu, 7->11->6 MB, 10 MB goal, 12 P

gc 1406 @6.070s 11%: 0.051+1.8+0.076 ms clock, 0.61+2.0/2.5/0+0.91 ms cpu, 8->11->6 MB, 13 MB goal, 12 P

gc 1407 @6.073s 11%: 0.052+1.8+0.20 ms clock, 0.62+1.5/2.2/0+2.4 ms cpu, 8->14->8 MB, 13 MB goal, 12 P
```

我们可以看到如何用GODEBUG变量生成go trace。
上面给出了3个具体的trace。

我们来看每一个变量的含义。

```
gc 1405 @6.068s 11%: 0.058+1.2+0.083 ms clock, 0.70+2.5/1.5/0+0.99 ms cpu, 7->11->6 MB, 10 MB goal, 12 P

// General
gc 1404     : The 1404 GC run since the program started
@6.068s     : Six seconds since the program started
11%         : Eleven percent of the available CPU so far has been spent in GC

// Wall-Clock
0.058ms     : STW        : Mark Start       - Write Barrier on
1.2ms       : Concurrent : Marking
0.083ms     : STW        : Mark Termination - Write Barrier off and clean up

// CPU Time
0.70ms      : STW        : Mark Start
2.5ms       : Concurrent : Mark - Assist Time (GC performed in line with allocation)
1.5ms       : Concurrent : Mark - Background GC time
0ms         : Concurrent : Mark - Idle GC time
0.99ms      : STW        : Mark Term

// Memory
7MB         : Heap memory in-use before the Marking started
11MB        : Heap memory in-use after the Marking finished
6MB         : Heap memory marked as live after the Marking finished
10MB        : Collection goal for heap memory in-use after Marking finished

// Threads
12P         : Number of logical processors or threads used to run Goroutines
```

我后面会涉及到大多数的值，但目前我们主要关注在trace 1405的内存部分。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure12.png)

```
// Memory
7MB         : Heap memory in-use before the Marking started
11MB        : Heap memory in-use after the Marking finished
6MB         : Heap memory marked as live after the Marking finished
10MB        : Collection goal for heap memory in-use after Marking finished
```

从上面个列表看到，在标记阶段开始前，堆里使用的内存是7MB；当标记完成时，使用的堆内存达到11MB。
这意味着，在回收时产生了额外4MB的内存。
当标记完成时，被标记为存活的堆内存是6MB。这也意味着，在下次gc开始前，程序的堆内存可以增长到12MB（假设gc百分比是100%）。

可以看到，回收器比它的目标差了1MB。在标记完成时，使用的堆内存是11MB，而不是10MB。
这是正常的，因为goal的计算是基于当前堆内存使用的数量、堆内存标记存活的数量、以及在回收时产生的额外分配。
在这个例子，应用在标记时，有时会比预期的需要更多的堆内存。

如果你看trace 1406，你会看到2ms后发生的情况。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure13.png)

```
gc 1406 @6.070s 11%: 0.051+1.8+0.076 ms clock, 0.61+2.0/2.5/0+0.91 ms cpu, 8->11->6 MB, 13 MB goal, 12 P

// Memory
8MB         : Heap memory in-use before the Marking started
11MB        : Heap memory in-use after the Marking finished
6MB         : Heap memory marked as live after the Marking finished
13MB        : Collection goal for heap memory in-use after Marking finished
```

从上面看到，在前一次gc后2ms，又开始了一次gc，尽管它使用的堆内存是8MB，而还没有达到允许的最大值12MB。
这是需要注意的地方，如果收集器决定提前开始回收会更好，那么它就会这么去做。
在这个case中，它提前开始的原因可能是，应用在分配的压力较大，收集器希望减少标记协助的延迟。

还有两个需要注意的地方。回收器这次达到了它的目标。标记完成后，在使用的堆内存是11MB，而不是13MB。
它标记为存活的堆内存同样是6MB。

此外，你可以使用gcpacertrace=1来获得更多的gc trace的信息。
这会让回收器打印concurrent pacer的内部状态信息。

```
$ export GODEBUG=gctrace=1,gcpacertrace=1 ./app

Sample output:
gc 5 @0.071s 0%: 0.018+0.46+0.071 ms clock, 0.14+0/0.38/0.14+0.56 ms cpu, 29->29->29 MB, 30 MB goal, 8 P

pacer: sweep done at heap size 29MB; allocated 0MB of spans; swept 3752 pages at +6.183550e-004 pages/byte

pacer: assist ratio=+1.232155e+000 (scan 1 MB in 70->71 MB) workers=2+0

pacer: H_m_prev=30488736 h_t=+2.334071e-001 H_T=37605024 h_a=+1.409842e+000 H_a=73473040 h_g=+1.000000e+000 H_g=60977472 u_a=+2.500000e-001 u_g=+2.500000e-001 W_a=308200 goalΔ=+7.665929e-001 actualΔ=+1.176435e+000 u_a/u_g=+1.000000e+000
```

运行gc trace可以帮助你了解应用的健康情况，以及回收器的速度。
回收器运营的速度对回收处理很重要。

#### 6、Pacing 

回收器有一个pacing算法来决定回收什么时候开始。
算法依赖于循环反馈（feedback loop），回收器用它来收集应用运行的信息，以及应用对堆的压力。
压力可以定义为，在给定的时间里，分配堆的速度。

在回收器开始回收时，它会计算它认为完成回收的时间。
当回收器开始运行时，延时会影响运行的应用程序。每一次回收都会增加应用的整体延迟。

一个误解是，减少回收器的频率能够改善性能。
它的想法是，如果你能延缓下一次回收的开始，在你可以延缓它造成的影响。但实时并非如此。

你可以把GC percentage改成超过100的值。
这会增加在下次回收时能否分配的堆内存。这会降低回收的pace。
不要这么做。（原文是：Don’t consider doing this.）

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure14.png)

上图展示了，改变GC Percentage会改变下次回收开始前允许分配的堆内存。
你可以看到，回收器是如何变慢的，因为它可以等待更多堆内存被使用。

通过直接影响回收的pace并不能优化回收器。这意味着每次回收之间，以及回收过程中，需要做更多的事情。
你可以通过减少添加到堆内存的数量和次数来影响它。

注意：这个想法是，你需要最少的堆来实现吞吐。
记住，当你在云环境运行时，最小化地使用像堆内存这样的资源，是很重要的。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure15.png)

上图展示了，本系列中下一部分要使用的一些go运行程序的数据统计。
在有10K请求通过应用程序处理时，蓝色部分展示没有进行任何优化的统计，而绿色则是有4.48G的non-productive被发现和移除。

看上面两个case的平均pace（2.08ms vs 1.96ms）。
它们看起来几乎是一样的，而其本质的区别在于，每次回收之间的工作量不同。
应用程序每次收集时处理的请求从7.13提高到3.98，这已经是79.1%的提高。
你可以看到，随着分配的减少，收集并没有变慢，而是保持一样。
而每次回收之间做更多工作的那个，显然更好一些。

调整回收的pace，以延迟delay的代价并不是改进你应用的性能的方法。
减少回收器运行的时间，其实就是降低了延迟的影响。
回收带来的延迟成本，前面已经介绍过了，现在来总结一下。

#### 7、回收器延迟代价

每次回收都会给你的应用程序带来两种延迟。
第一种是占用CPU容量。它窃取了CPU容量意味着你的应用程序在回收时并没有全速运行。
应用程序的goroutine会与回收器的goroutine共享P，或者帮助回收（标记协助）。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure16.png)

上图显示了，应用程序只使用了CPU容量的75%。
因为回收器独占了P1。大多数回收器都是这样的。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure17.png)
上图显示了，应用程序这时只占了CPU容量的50%。
这是因为在P3上的goroutine运行标记协助（Mark Assist），而P1又被回收器独占。

注意：标记1MB的live对内存通常需要4个CPU-milliseconds。
（例如，要计算花费多少浩渺，只需要把live堆内存转成MB单位，然后除以0.25*CPU数量）
标记的速度实际上是1MB/s，但它实际只占用1/4的CPU。

第二种延迟是在回收时的STW延迟。
执行STW时，是没有任何应用程序的goroutine执行操作的。
应用程序也停止了。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-gc-part1_figure18.png)

上图显示了，STW延迟时，所有的goroutine都停止了。
这在每次回收时会发生2次。
如果你的应用是健康的，大多数回收器能把STW时间保持在100毫秒以内。

我们已经知道，在回收的不同阶段，内存如何分配，pacing如何工作，以及回收器对运行的程序造成的不同延迟。
有了这些知识，就能对回收器进行优化了。

#### 8、优化

对回收器的优化是减少堆内存的压力。
记住，压力可以定义为，应用在给定的时间内，能够多快地分配堆内存。
当压力减少时，由回收导致的延迟也会减少。GC延迟是导致你的应用减慢的原因。

减少GC延迟的方式是，定义并移除你的应用中没有必要的内存分配。
这些方法或许能帮到你：

* 保存可能的最少的堆

* 找到优化的pace

* 每次分配都要有明确的目标

* 最小化每次的回收、STW和Mark Assist。

这些能够帮助减少回收器对你运营应用造成的延迟，进而增加应用程序的性能和吞吐量。
它对回收的pace没有影响。你可以做其他的工程决策来减少堆的压力。

理解你的应用程序工作量的实质。
理解你的工作量意味着你需要合理地使用goroutine的数量。
CPU和IO带宽意味着需要做不同的工程决策。

https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html

理解定义的数据，以及它如何在应用之间传递的。
这意味着你需要理解你要解决的问题的数据。
数据一致性是保持数据完整性的重要部分，当你选择在栈上进行堆分配（heap allocation）时，它能帮助你。

https://www.ardanlabs.com/blog/2017/06/design-philosophy-on-data-and-semantics.html







