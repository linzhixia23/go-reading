原文:https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html

#### 1、介绍

当我解决一个问题，尤其是新的问题，我一开始不会考虑这个问题是否能并发。
我会先找一个简单的方法，并证明它是对的；之后我才开始问自己，如果用并发是否是合理并且实际的。
有时，并发是合适的，有时它并非那么显而易见。

在第1章中，我介绍了OS调度的机制，这对于你开始写多线程的代码是很重要的。
第2章中，我解释了go的调度，这对于理解go的并发代码很重要。
这一篇里，我会把OS的调度机制和go的调度机制结合起来，并深度理解并发是什么，以及不是什么。

这篇文章的目标是：

* 提供基本的semantics，帮助你决定一个工作是否适合使用并发

* 展示不同类型的工作如何改变semantics，以及你如何做出工程的选择

#### 2、什么是并发

并发意味这"乱序"执行。给定一系列的指令，并找到一个打乱原先顺序的执行序列，并能保证跟之前的顺序得到同样的结果。
那么，你面对的问题是，显然乱序执行要能产生价值。
这里说的价值，指的是获得性能的收益。
根据你的问题，乱序执行可能并不实际，甚至毫无意义。


[理解并发不是并行](https://blog.golang.org/waza-talk)是很重要的。
并行意味着同时执行2个或更多指令。
它跟并发的概念不同。并行只有在你有至少2个操作系统和硬件线程，并且你有至少2个goroutine，每一个相互独立地执行在OS/硬件线程上。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-scheduler-concurrency_figure01.png)

上图中，有两个逻辑处理器（P），它们每一个都有相互独立的OS线程(M)附加在硬件线程（core）。
这时，两个goroutine（G1和G2）是并行执行的。
在每个逻辑处理器上，3个goroutine在它们各自的OS线程上交互执行。
所有这些goroutine是并发执行的，执行它们的指令在OS线程上，并没有特定的顺序和共享时间。

这就是产生摩擦的地方：有时不用并行的方式来使用并发，实际上会降低你的吞吐量。
有意思的地方是，有时以并行的方式来使用并不能像你想的那样获得较大的性能。

#### 3、工作量

你知道乱序执行是怎么样的吗？ 首先，你要理解问题的workload。
当考虑并发的时候，有两类workload比较重要：

* CPU-Bound。这类workload不会让gorouine等待。它会持续进行计算。一个计算Pi的第N位的线程是CPU-Bound。

* IO-Bound。这类workload会导致goroutine进入等待状态。这些工作包括通过网络请求资源，系统调用，等到时间返回。goroutine在等待读文件也是IO-Bound。
在这里，我会介绍一些导致goroutine等待的异步事件（mutexes，atomic）。

对于CPU-Bound，你需要并行来使用并发。单个OS/核来处理多个goroutine并不高效，因为需要换入换出。
这时，goroutine越多就越会降低执行的工作量，因为把goroutine换入换出OS线程会有延迟代价。
上下文切换会导致你的工作"Stop The World"，因为这时你任何的工作都不会执行。

对于IO-Bound，你并不需要并行来使用并发。单个OS/核能够处理多个goroutine，因为goroutine的工作中会存在等待状态。
当goroutine大于OS/核时，能够加速工作量的执行，因为goroutine的换入换出的延迟代价并不会导致你的工作停滞。
当你的工作停止时，它可以让另一个goroutine使用同样的OS/核，而不是让OS/核进入idle状态。

如何知道每个核上挂多少个goroutine能够达到最好的吞吐量呢。
如果goroutine数量少，则你会有很多idle时间；如果goroutine数量多，则切换上下文的耗时会变多。
这也是你在实际设计中应该考量的。

现在，我们来看一些代码帮助你了解何时能使用并发，以及是否需要并行。

#### 4、数字求和

我们来看一个代码，它是add一些整数。

https://play.golang.org/p/r9LdqUsEzEz
```go
36 func add(numbers []int) int {
37     var v int
38     for _, n := range numbers {
39         v += n
40     }
41     return v
42 }
```

在36行，定义了一个add函数，它对集合里的数求和，并返回结果。
在37行，定义了变量v来存储和。
在38行，函数线性遍历集合，并在39行求和。最终41行返回结果。

问题是：add函数是否合适用于乱序执行？我相信答案是yes。
集合里的整数可以分成若干小的list，这些list可以并发执行。
当所有小的list求和之后，它们能够得到跟顺序执行版本一样的结果。

那么，下一个问题是：要分成多少个小的list，才能得到最好的吞吐量。
要回答这个问题，你需要了解add函数是那种类型的工作量（workload）。
add函数是CPU-Bound，因为算法是纯粹的数字运算，而且不会让goroutine进入等待状态。
这意味着用每个OS/核用一个goroutine能够获得好的吞吐量。

以下是一个add的并发版本。

https://play.golang.org/p/r9LdqUsEzEz
```go
44 func addConcurrent(goroutines int, numbers []int) int {
45     var v int64
46     totalNumbers := len(numbers)
47     lastGoroutine := goroutines - 1
48     stride := totalNumbers / goroutines
49
50     var wg sync.WaitGroup
51     wg.Add(goroutines)
52
53     for g := 0; g < goroutines; g++ {
54         go func(g int) {
55             start := g * stride
56             end := start + stride
57             if g == lastGoroutine {
58                 end = totalNumbers
59             }
60
61             var lv int
62             for _, n := range numbers[start:end] {
63                 lv += n
64             }
65
66             atomic.AddInt64(&v, int64(lv))
67             wg.Done()
68         }(g)
69     }
70
71     wg.Wait()
72
73     return int(v)
74 }
```

上面的addConcurrent是add函数的并发版本。
它比前面的代码要复杂一些，所以我会重点讲其中的要点。

48行，每个goroutine获得各自的相对小的list来相加。它的大小是根据数组的长度和goroutine的数量计算的。

53行，创建goroutine池来完成相加的任务。

57-59行，最后一个goroutine会相加剩下的数字，因而它可能比其他goroutine的要多一些。

66行，把每个goroutine计算的结果加到最终的结果。

并发的版本比顺序执行的版本更复杂，但是这样增加复杂度是否值得呢？
回答这个问题最好的方式就是创建一个benchmanrk。
在benchmark中，我使用1千万的数字，并且关闭了gc。

```go
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(numbers)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        addConcurrent(runtime.NumCPU(), numbers)
    }
}
```

上面展示了benchmark函数。顺序执行的版本只会用1个goroutine，而并发执行的版本会使用runtime.NumCPU()个goroutine（在我的机器上是8个）。
在这个例子中，并发版本使用了并发而没有并行（ leveraging concurrency without parallelism）。


```
10 Million Numbers using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential      	    1000	   5720764 ns/op : ~10% Faster
BenchmarkConcurrent      	    1000	   6387344 ns/op
BenchmarkSequentialAgain 	    1000	   5614666 ns/op : ~13% Faster
BenchmarkConcurrentAgain 	    1000	   6482612 ns/op
```

注意：在你的本地机器运行benchmark很复杂。有很多因素会导致你的benchmark不准确。
你需要确保你的机器尽可能的空闲，并多运行benchmark几次。

benchmark的结果显示，串行版本比并发版本快10-13%。
并发版本会在单个OS线程上进行上下文切换，并进行goroutine的管理。

以下是一个单独的OS/核能给每一个goroutine使用的结果。
顺序执行版本使用1个goroutine，而并发版本使用run.NumCPU个goroutine。
在这个例子中，并发版本使用了并行的并发（ leveraging concurrency with parallelism）

```
10 Million Numbers using 8 goroutines with 8 cores
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 8 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential-8        	    1000	   5910799 ns/op
BenchmarkConcurrent-8        	    2000	   3362643 ns/op : ~43% Faster
BenchmarkSequentialAgain-8   	    1000	   5933444 ns/op
BenchmarkConcurrentAgain-8   	    2000	   3477253 ns/op : ~41% Faster
```

从上面可以看到，当每个goroutine使用一个独立的OS/核的情况下，并发版本比串行版本快41-43%。
这时所有的goroutine是并行执行的，8个goroutine在相同的时间里会做相同的任务。

#### 5、排序

需要注意的是，并不是所有的CPU-Bound任务都适用于并发。
尤其是对于拆解并合并结果的代价很大的任务。
一个例子是冒泡排序。下面看下其go实现版本。

https://play.golang.org/p/S0Us1wYBqG6
```go
01 package main
02
03 import "fmt"
04
05 func bubbleSort(numbers []int) {
06     n := len(numbers)
07     for i := 0; i < n; i++ {
08         if !sweep(numbers, i) {
09             return
10         }
11     }
12 }
13
14 func sweep(numbers []int, currentPass int) bool {
15     var idx int
16     idxNext := idx + 1
17     n := len(numbers)
18     var swap bool
19
20     for idxNext < (n - currentPass) {
21         a := numbers[idx]
22         b := numbers[idxNext]
23         if a > b {
24             numbers[idx] = b
25             numbers[idxNext] = a
26             swap = true
27         }
28         idx++
29         idxNext = idx + 1
30     }
31     return swap
32 }
33
34 func main() {
35     org := []int{1, 3, 2, 4, 8, 6, 7, 2, 3, 0}
36     fmt.Println(org)
37
38     bubbleSort(org)
39     fmt.Println(org)
40 }
```

问题：冒泡排序是否适合用乱序执行。我相信答案是no。
整数能够分成若干个小的list，并发进行排序。
然而，当所有的并发工作完成时，并没有一个高效的方案能够合并这些结果。
以下是冒泡排序的并发版本。

```go
01 func bubbleSortConcurrent(goroutines int, numbers []int) {
02     totalNumbers := len(numbers)
03     lastGoroutine := goroutines - 1
04     stride := totalNumbers / goroutines
05
06     var wg sync.WaitGroup
07     wg.Add(goroutines)
08
09     for g := 0; g < goroutines; g++ {
10         go func(g int) {
11             start := g * stride
12             end := start + stride
13             if g == lastGoroutine {
14                 end = totalNumbers
15             }
16
17             bubbleSort(numbers[start:end])
18             wg.Done()
19         }(g)
20     }
21
22     wg.Wait()
23
24     // Ugh, we have to sort the entire list again.
25     bubbleSort(numbers)
26 }
```

上述代码中，使用了多个goroutine来对list的部分进行排序。
然而，你得到的只是多个部分有序的chunk。
例如，有36个数字，你把每12数字分成一组，那么执行到25行时，它的结果是这样的。

```
Before:
  25 51 15 57 87 10 10 85 90 32 98 53
  91 82 84 97 67 37 71 94 26  2 81 79
  66 70 93 86 19 81 52 75 85 10 87 49

After:
  10 10 15 25 32 51 53 57 85 87 90 98
   2 26 37 67 71 79 81 82 84 91 94 97
  10 19 49 52 66 70 75 81 85 86 87 93
```

冒泡排序的本质是交换list的元素，而在25行时，没有办法通过并发提高性能。
因而，对于冒泡排序发，并发不会带来任何收益。

#### 6、读取文件 

上面已经展示两个CPU-Bound的例子，那么对于IO-Bound是什么情况呢。
当goroutine换入换出时有等待状态，情况是否会不一样？
下面来看一个IO-Bound的例子，它从文件读取，并进行文本搜索。

下面来实现一个串行版本：find函数。

https://play.golang.org/p/8gFe5F8zweN
```go
42 func find(topic string, docs []string) int {
43     var found int
44     for _, doc := range docs {
45         items, err := read(doc)
46         if err != nil {
47             continue
48         }
49         for _, item := range items {
50             if strings.Contains(item.Description, topic) {
51                 found++
52             }
53         }
54     }
55     return found
56 }
```

43行，定义了found变量，来统计某个特定的主题在指定文本出现的次数。
44行，迭代遍历文档，并通过45行的read函数读问文件。
49-53行，contain函数会查看主题是否在文档中。如果在的话，found变量会增加1。

以下是read函数的实现版本。
 
https://play.golang.org/p/8gFe5F8zweN
```go
33 func read(doc string) ([]item, error) {
34     time.Sleep(time.Millisecond) // Simulate blocking disk read.
35     var d document
36     if err := xml.Unmarshal([]byte(file), &d); err != nil {
37         return nil, err
38     }
39     return d.Channel.Items, nil
40 }
```

read函数开始的时候会sleep，以模拟我们进行系统调用的延时。这个延迟的一致性对于我们衡量串行版本和并行版本的性能很重要。
在35-39行，模拟的xml文件会unmarshal成struct来进行处理。最终，39行会返回所有值。

以下是并发版本

https://play.golang.org/p/8gFe5F8zweN
```go
58 func findConcurrent(goroutines int, topic string, docs []string) int {
59     var found int64
60
61     ch := make(chan string, len(docs))
62     for _, doc := range docs {
63         ch <- doc
64     }
65     close(ch)
66
67     var wg sync.WaitGroup
68     wg.Add(goroutines)
69
70     for g := 0; g < goroutines; g++ {
71         go func() {
72             var lFound int64
73             for doc := range ch {
74                 items, err := read(doc)
75                 if err != nil {
76                     continue
77                 }
78                 for _, item := range items {
79                     if strings.Contains(item.Description, topic) {
80                         lFound++
81                     }
82                 }
83             }
84             atomic.AddInt64(&found, lFound)
85             wg.Done()
86         }()
87     }
88
89     wg.Wait()
90
91     return int(found)
92 }
```

并发版本用了30行代码，而串行版本只有13行。我的实现并发版本中，由于要处理的文章量不知道，因而要控制goroutine的数量。
因而，我选择使用一个channel池来往goroutine池里feed要处理文章。

以下是代码中比较重要的部分：

第61-64行：创建channel，写入所有要处理的文档。

第65行：关闭channel，所有goroutine会在文档处理完后结束。

第70行：创建goroutine池

第73-83行：每个池里的goroutine会从channel接收文档，读入内存，并检查它的topic。
如果匹配，则本地变量会增加。

第84行：把单个goroutine计算的结果求和，得到最终的结果。

显然，并发版本比串行版本复杂，那么它是否值得呢？
当然，最好的方式是benchmark。在benchmark中，我使用1000个文档，并关闭了gc。

```go
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        find("test", docs)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        findConcurrent(runtime.NumCPU(), "test", docs)
    }
}
```

我们来看多个goroutine共用1个OS/核线程的结果。
在这个版本中，并发版本没有并行。

```
10 Thousand Documents using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/io-bound
BenchmarkSequential      	       3	1483458120 ns/op
BenchmarkConcurrent      	      20	 188941855 ns/op : ~87% Faster
BenchmarkSequentialAgain 	       2	1502682536 ns/op
BenchmarkConcurrentAgain 	      20	 184037843 ns/op : ~88% Faster
```

benchmark结果显示，并发版本比并行版本大约快87-88%。
这是因为goroutine共享单个OS/核线程会更加高效。上下文切换的时候，它允许更多任务在单个线程上执行。

下面是并行版本。

```
10 Thousand Documents using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/io-bound
BenchmarkSequential-8        	       3	1490947198 ns/op
BenchmarkConcurrent-8        	      20	 187382200 ns/op : ~88% Faster
BenchmarkSequentialAgain-8   	       3	1416126029 ns/op
BenchmarkConcurrentAgain-8   	      20	 185965460 ns/op : ~87% Faster
```

benchmark结果显示，增加额外的线程并没有带来更多性能的提升。

#### 7、结论

这篇文章的目标是指引你如何考虑是否适合使用并发。
我使用了不同的算法和workload，这样你能看到不同的场景，以及不同的工程决定。

你可以看到，在IO-Bound的情况下，并行并不会带来大的性能提升。
而对于CPU-Bound，情况则刚好相反。
而对于像冒泡排序这样的算法，使用并发只会增加复杂度，而不会有任何性能的提高。
因此，重要的是，你要看你的workload是否适合使用并发，然后决定根据workload的类型来选择合适的方案。

